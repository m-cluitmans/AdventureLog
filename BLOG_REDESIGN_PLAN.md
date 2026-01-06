# AdventureLog Blog View Redesign Plan

**Created:** 2026-01-06
**Purpose:** Transform AdventureLog into a story-based travel diary suitable for sharing sabbatical experiences with family and friends

---

## 🎯 Project Goals

### Primary Objective
Create a parallel blog-style view for AdventureLog collections that presents travel experiences as a narrative story rather than data tiles, optimized for a 14-week sabbatical journey.

### Key Requirements
1. **Keep existing AdventureLog UI** - Maintain current data management interface
2. **Add new blog view** - Create public-facing story presentation
3. **Paginated navigation** - Handle long trips (14 weeks) with day-by-day or post-by-post (where a post can contain multiple elements, such as locations, transportation, markdown note, etc) pagination
4. **Smart navigation** - Remember viewer's last position, highlight new content
5. **Full-width media** - Hero images and photo galleries for visual storytelling
6. **Secret public URLs** - Shareable links not indexed by search engines
7. **Mobile-friendly** - Responsive design for viewing on all devices

---

## 📐 Architecture Design

### Explicit Post Model (First-Class Entities)

#### Content Organization Philosophy

**Post = First-Class Entity**
- **Post** is a database model (not virtual grouping)
- Each Post has: title, date range, hero image, published status
- Posts explicitly link to content items (Notes, Locations, Transportation, etc.)
- Content items can belong to multiple posts or none
- Content order within post is explicit (via `order` field)

**Collection = Trip/Sabbatical**
- Collection contains multiple Posts
- Collection has overall date range
- Share URL shows all posts in trip: `/share/{collection-id}`

**Example Structure:**
```
Collection: "Sabbatical 2026" (Jan 2 - June 15, 2026)
├── Post 1: "Arrival in Granada" (Jan 4-6)
│   ├── [Order 1] Note: "Three Days in Granada"
│   ├── [Order 2] Transportation: Car journey
│   ├── [Order 3] Note: "Exploring Albaicín"
│   ├── [Order 4] Location: Albaicín neighborhood
│   ├── [Order 5] Note: "The Alhambra"
│   ├── [Order 6] Location: Alhambra
│   └── [Order 7] Location: Mirador de San Nicolás
├── Post 2: "Day Trip to Sierra Nevada" (Jan 7)
│   ├── [Order 1] Note: "Mountain adventure"
│   ├── [Order 2] Transportation: Car
│   └── [Order 3] Location: Sierra Nevada
└── Post 3: "Coastal Route" (Jan 8-10)
    ├── [Order 1] Note: "Driving along the coast"
    ├── [Order 2] Transportation: Car
    ├── [Order 3] Location: Málaga
    └── [Order 4] Location: Nerja
```

#### New Routes Structure
```
/share/[collection_id]                    → Trip overview (lists all posts)
/share/[collection_id]/[post_slug]        → Individual post view
/share/[collection_id]/map                → Full trip map view
```

**Navigation:**
- Trip overview shows all posts with preview
- Post navigation: Previous/Next post buttons
- Progress indicator: "Post 3 of 25"
- Remember last viewed post (localStorage)

#### Component Hierarchy
```
ShareLayout.svelte (wrapper for all blog views)
├── ShareHeader.svelte (trip title, date range, navigation)
├── TripOverview.svelte (NEW - landing page showing all posts)
│   └── PostPreviewCard.svelte (post card with title, date, hero image)
├── PostNavigator.svelte (pagination, progress bar, "new content" badges)
├── PostView.svelte (main post content renderer)
│   ├── PostHeader.svelte (title, date range)
│   ├── HeroGallery.svelte (full-width photo display)
│   ├── MixedContentStream.svelte (renders all post items in order)
│   │   ├── NoteSection.svelte (markdown note)
│   │   ├── LocationStoryCard.svelte (location with details)
│   │   ├── TransportationStory.svelte (journey details)
│   │   ├── PhotoGallery.svelte (photo grid)
│   │   ├── LodgingCard.svelte (accommodation details)
│   │   └── ChecklistDisplay.svelte (checklist items)
│   ├── RouteMap.svelte (embedded map for post locations)
│   └── PostFooter.svelte (prev/next post navigation)
└── ShareFooter.svelte (trip stats, social sharing)
```

---

## 🔧 Data Model & Backend Changes

### New Models Required

**1. Post Model**
```python
# backend/server/adventures/models.py

class Post(models.Model):
    """A blog post within a collection (trip)"""
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    collection = models.ForeignKey(Collection, on_delete=models.CASCADE,
                                   related_name='posts')
    user = models.ForeignKey(User, on_delete=models.CASCADE)

    # Post metadata
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, blank=True)  # For nice URLs
    start_date = models.DateField()
    end_date = models.DateField(null=True, blank=True)  # Can be same as start_date

    # Display settings
    hero_image = models.ForeignKey('ContentImage', null=True, blank=True,
                                   on_delete=models.SET_NULL,
                                   related_name='post_hero')
    is_published = models.BooleanField(default=False)  # Draft vs. published
    order = models.IntegerField()  # Order within collection (1, 2, 3...)

    # Timestamps
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['collection', 'order']
        unique_together = [['collection', 'slug']]
        indexes = [
            models.Index(fields=['collection', 'order']),
            models.Index(fields=['collection', 'is_published']),
        ]

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

**2. PostItem Model (Links content to posts)**
```python
class PostItem(models.Model):
    """Many-to-many relationship between Post and content items with ordering"""
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='items')

    # Polymorphic content reference (can link to any content type)
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.UUIDField()
    content_object = GenericForeignKey('content_type', 'object_id')

    # Order within post
    order = models.IntegerField()

    # Timestamps
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['post', 'order']
        unique_together = [['post', 'content_type', 'object_id']]
        indexes = [
            models.Index(fields=['post', 'order']),
        ]
```

**3. Update existing models to add reverse relations**
```python
from django.contrib.contenttypes.fields import GenericRelation

class Note(models.Model):
    # ... existing fields ...
    post_items = GenericRelation('PostItem')

class Visit(models.Model):
    # ... existing fields ...
    post_items = GenericRelation('PostItem')

class Transportation(models.Model):
    # ... existing fields ...
    post_items = GenericRelation('PostItem')

class Lodging(models.Model):
    # ... existing fields ...
    post_items = GenericRelation('PostItem')

class Checklist(models.Model):
    # ... existing fields ...
    post_items = GenericRelation('PostItem')
```

### Migrations

```bash
# Create migrations
python manage.py makemigrations adventures

# Apply migrations
python manage.py migrate adventures
```

### API Serializers

**Post Serializer:**
```python
# backend/server/adventures/serializers.py

class PostItemSerializer(serializers.ModelSerializer):
    """Serializes a post item with its linked content"""
    content_type = serializers.SerializerMethodField()
    content = serializers.SerializerMethodField()

    class Meta:
        model = PostItem
        fields = ['id', 'order', 'content_type', 'content']

    def get_content_type(self, obj):
        """Return content type name"""
        return obj.content_type.model

    def get_content(self, obj):
        """Return serialized content object"""
        content = obj.content_object
        if isinstance(content, Note):
            return NoteSerializer(content).data
        elif isinstance(content, Visit):
            return VisitSerializer(content).data
        elif isinstance(content, Transportation):
            return TransportationSerializer(content).data
        elif isinstance(content, Lodging):
            return LodgingSerializer(content).data
        elif isinstance(content, Checklist):
            return ChecklistSerializer(content).data
        return None


class PostSerializer(serializers.ModelSerializer):
    """Serializes a blog post with all its items"""
    items = PostItemSerializer(many=True, read_only=True)
    hero_image_url = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = ['id', 'collection', 'user', 'title', 'slug', 'start_date',
                 'end_date', 'hero_image', 'hero_image_url', 'is_published',
                 'order', 'items', 'created_at', 'updated_at']
        read_only_fields = ['slug']

    def get_hero_image_url(self, obj):
        if obj.hero_image:
            return obj.hero_image.image.url
        return None


class CollectionWithPostsSerializer(serializers.ModelSerializer):
    """Serializes a collection with all its posts"""
    posts = serializers.SerializerMethodField()

    class Meta:
        model = Collection
        fields = ['id', 'name', 'description', 'start_date', 'end_date',
                 'is_public', 'posts', 'created_at', 'updated_at']

    def get_posts(self, obj):
        """Return only published posts for public sharing"""
        posts = obj.posts.filter(is_published=True).order_by('order')
        return PostSerializer(posts, many=True).data
```

### API Endpoints

**New Endpoints:**
```python
# GET /api/collections/{id}/posts/
# Returns all posts in a collection (filtered by is_published for non-owners)

# POST /api/collections/{id}/posts/
# Create a new post

# GET /api/posts/{id}/
# Get a specific post with all its items

# PATCH /api/posts/{id}/
# Update post metadata (title, dates, hero_image, is_published)

# DELETE /api/posts/{id}/
# Delete a post

# POST /api/posts/{id}/items/
# Add content items to post
# Body: [{ content_type: 'note', object_id: 'uuid', order: 1 }, ...]

# PATCH /api/posts/{id}/items/reorder/
# Reorder items within post
# Body: [{ id: 'post_item_id', order: 1 }, ...]

# DELETE /api/posts/{post_id}/items/{item_id}/
# Remove content item from post
```

### Frontend Data Flow

**TypeScript Interfaces:**
```typescript
interface Post {
  id: string;
  collection: string;
  user: string;
  title: string;
  slug: string;
  start_date: string;
  end_date: string | null;
  hero_image: string | null;
  hero_image_url: string | null;
  is_published: boolean;
  order: number;
  items: PostItem[];
  created_at: string;
  updated_at: string;
}

interface PostItem {
  id: string;
  order: number;
  content_type: 'note' | 'visit' | 'transportation' | 'lodging' | 'checklist';
  content: Note | Visit | Transportation | Lodging | Checklist;
}

interface CollectionWithPosts {
  id: string;
  name: string;
  description: string;
  start_date: string;
  end_date: string;
  is_public: boolean;
  posts: Post[];
}
```

**Fetching Data:**
```typescript
// In +page.server.ts
export async function load({ params }) {
  const response = await fetch(
    `${API_URL}/api/collections/${params.id}/posts/`
  );
  const posts: Post[] = await response.json();

  return {
    collection: {
      /* ... */
    },
    posts
  };
}
```

### Multi-Day Post Example

**Scenario:** 3-day Granada visit with flexible content ordering

**Step 1: User Creates Content in AdventureLog**
```
Collection: "Sabbatical 2026"

Note 1: "Three Days in Granada"
  Content: "Our Granada adventure started..."
  Date: Jan 4, 2026

Note 2: "Exploring Albaicín"
  Content: "First evening we wandered..."
  Date: Jan 4, 2026

Note 3: "The Alhambra"
  Content: "Day 2 we visited the palace..."
  Date: Jan 5, 2026

Note 4: "Sunset Viewpoints"
  Content: "Last day was more relaxed..."
  Date: Jan 6, 2026

Location: Albaicín neighborhood
  Visit: Jan 4, 3pm-7pm

Location: Alhambra
  Visit: Jan 5, 10am-2pm

Location: Mirador de San Nicolás
  Visit: Jan 6, 5pm-7pm

Transportation: Car
  Netherlands → Granada
  Jan 2-4, 2026

Lodging: Hotel Casa del Capitel Nazarí
  Jan 4-7, 2026
```

**Step 2: User Creates a Post**
```
Click "Create Blog Post" in Collection view

Post Form:
  Title: "Three Days in Granada"
  Start Date: Jan 4, 2026
  End Date: Jan 6, 2026
  Hero Image: [Select from Alhambra photos]
  Status: Published
```

**Step 3: User Adds Content to Post (Drag & Drop)**
```
Available Content (Jan 4-6):    →    Post Items:
□ Note 1: "Three Days..."            ☰ [1] Note: "Three Days..."
□ Note 2: "Exploring..."             ☰ [2] Transportation: Car
□ Note 3: "The Alhambra"             ☰ [3] Note: "Exploring..."
□ Note 4: "Sunset..."                ☰ [4] Location: Albaicín
□ Location: Albaicín                 ☰ [5] Note: "The Alhambra"
□ Location: Alhambra                 ☰ [6] Location: Alhambra
□ Location: Mirador                  ☰ [7] Note: "Sunset..."
□ Transportation: Car                ☰ [8] Location: Mirador
□ Lodging: Hotel                     ☰ [9] Lodging: Hotel
```

**Step 4: Blog View Displays**
```
URL: /share/sabbatical-2026/three-days-in-granada

Post: "Three Days in Granada"
Date Range: Jan 4-6, 2026 (3 days)

[Hero Image: Alhambra photo]

---

## Three Days in Granada
Our Granada adventure started with a long drive from the Netherlands...

🚗 Car - Netherlands to Granada
1,746 km | Jan 2-4
[Photos from journey]

---

## Exploring Albaicín
First evening we wandered through the narrow streets...

📍 Albaicín neighborhood
⭐⭐⭐⭐⭐ | Visited Jan 4
[Photo gallery from Albaicín]

---

## The Alhambra
Day 2 we finally visited the palace. It was stunning!

📍 Alhambra
⭐⭐⭐⭐⭐ | Visited Jan 5
[Photo gallery from Alhambra]

---

## Sunset Viewpoints
Last day was more relaxed with sunset at Mirador...

📍 Mirador de San Nicolás
⭐⭐⭐⭐⭐ | Visited Jan 6
[Photo gallery from viewpoint]

---

🏨 Hotel Casa del Capitel Nazarí
⭐⭐⭐⭐ | Jan 4-7
[Lodging photos]

[Map showing all locations]

← Previous Post    |    Next Post →
```

**Key Points:**
- Post is created explicitly, not automatically
- User decides title, date range, and what content to include
- Content can be reordered by dragging within post
- Same content can appear in multiple posts (e.g., hotel spans multiple posts)
- Natural narrative flow: text → journey → text → location → text → location

### Benefits of Explicit Post Model

✅ **Clear ownership** - Content explicitly belongs to posts
✅ **Full manual control** - Drag-and-drop any content in any order within post
✅ **Flexible post length** - Can be 1 day or many days (you decide)
✅ **Content reuse** - Same location can appear in multiple posts
✅ **Mixed content** - Interleave notes, locations, photos, transportation
✅ **Blog-like flow** - Text → photo → text → map → text (any sequence)
✅ **Draft/publish workflow** - Posts can be drafts before publishing
✅ **No automatic grouping** - No confusing rules, you create posts explicitly
✅ **Clean URLs** - `/share/trip-name/post-slug` (SEO-friendly)
✅ **First-class entity** - Posts are database models, not virtual constructs

---

## 🗺️ User Experience Flow

### For Content Creator (You)

**Phase 1: Create Content** (as you travel)
1. Use existing AdventureLog interface to add:
   - Notes with markdown stories
   - Locations with photos
   - Transportation details
   - Lodging information
2. This works exactly as before - no change to existing workflow

**Phase 2: Create Blog Posts** (weekly or after trip)
1. Go to Collection page
2. Click "Create Blog Post" button
3. Fill in post form:
   - Title: "Three Days in Granada"
   - Date range: Jan 4-6, 2026
   - Select hero image
   - Status: Draft or Published
4. Drag content items into post in desired order
5. Preview blog view
6. Publish post

**Phase 3: Share Sabbatical** (once)
1. Mark collection as `is_public = true`
2. Click "Share Sabbatical" button
3. Copy share link: `https://yourserver.com/share/sabbatical-2026`
4. Send link to family/friends

### For Viewers (Family/Friends)

**First Visit:**
1. Open share link: `https://yourserver.com/share/sabbatical-2026`
2. See trip overview page:
   - Hero image
   - Trip title: "Sabbatical 2026"
   - Date range: Jan 2 - June 15, 2026
   - Grid of all published posts with preview cards
3. Click a post to read it
4. Navigate between posts with Previous/Next buttons

**Reading a Post:**
1. See post title and date range
2. View hero image
3. Scroll through mixed content:
   - Notes (text sections)
   - Photos (galleries)
   - Locations (with details and maps)
   - Transportation (journey cards)
4. Map at bottom shows all locations in post
5. Navigate to next post or back to trip overview

**Return Visits:**
1. Browser remembers last viewed post (localStorage)
2. Landing page shows "Continue Reading: Post 5"
3. Badge shows "New: 3 posts added"
4. Can jump to specific post from overview

**Navigation Options:**
- Trip overview grid (see all posts)
- Previous/Next buttons (sequential reading)
- Progress indicator: "Post 3 of 25"
- Full trip map (all locations from all posts)

---

## 🎨 Visual Design Specification

### Landing Page (`/share/[collection_id]`)
```
┌────────────────────────────────────────────┐
│  [Full-width hero image from trip]         │
│                                            │
│     SABBATICAL 2026                        │
│     Netherlands → Spain → ...              │
│     January 2 - June 15, 2026              │
│     97 days | 15 countries                 │
│                                            │
│  [Start Reading from Day 1]  [View Map]    │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│  Trip Overview                             │
│  [Generated description from collection]   │
│                                            │
│  Quick Navigation:                         │
│  [Week 1: Days 1-7]  [Week 2: Days 8-14]  │
│  [Week 3: Days 15-21] ...                 │
└────────────────────────────────────────────┘
```

### Day View (`/share/[collection_id]/day/4`)
```
┌────────────────────────────────────────────┐
│  SABBATICAL 2026          [Day ▼]  [≡ Menu] │
│  ────────────────────────                  │
│  Day 4 of 97                               │
│  ●●●●○○○○○○○○○○○○ (progress bar)          │
│  ← Day 3          Granada, Spain   Day 5 → │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│  [Full-width hero image: Alhambra]         │
│  📷 1 / 8                                   │
│  [‹  ›] navigation arrows                  │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│  DAY 4 - Granada                           │
│  January 4, 2026                           │
│  📍 Granada, Andalucia, España             │
│  ⭐⭐⭐⭐⭐                                   │
├────────────────────────────────────────────┤
│                                            │
│  De eerste dag                             │
│  ─────────────────                         │
│                                            │
│  Vandaag lekker op reis. Jadiedatdie      │
│  datie.Vandaag lekker op reis...          │
│                                            │
│  [Full markdown-rendered note content]     │
│  - Paragraphs with spacing                │
│  - Inline images if any                   │
│  - Headers, lists, etc.                   │
│                                            │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│  [Photo Gallery Grid]                      │
│  ┌──────┐ ┌──────┐ ┌──────┐               │
│  │ img  │ │ img  │ │ img  │               │
│  └──────┘ └──────┘ └──────┘               │
│  ┌──────┐ ┌──────┐ ┌──────┐               │
│  │ img  │ │ img  │ │ img  │               │
│  └──────┘ └──────┘ └──────┘               │
│  [Click to expand to lightbox]             │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│  [Embedded map showing Granada location]   │
│  With marker and optional route           │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│  Journey to Next Destination               │
│  🚗 Car                                    │
│  Granada → Seville                         │
│  1,746 km | Jan 2-4, 2026                 │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│         ← Day 3          Day 5 →           │
│                                            │
│  [Back to Trip Overview]                   │
└────────────────────────────────────────────┘
```

### Mobile Responsive Adjustments
- Full-width images maintained
- Single column layout
- Sticky header with day navigation
- Swipe gestures for prev/next day
- Hamburger menu for day selector
- Collapsed gallery (2 columns instead of 3)

---

## 🔒 Privacy & Access Control

### Implementation Strategy

#### Secret URLs (Not Indexed)
```html
<!-- In share page <head> -->
<meta name="robots" content="noindex, nofollow">
<meta name="googlebot" content="noindex, nofollow">
```

#### URL Structure
- Use collection UUID (already random/unguessable)
- Example: `/share/7f3a8b2c-4d1e-9f6a-8c2b-3e5f7a9b1c4d`
- No need for additional tokens (UUID is sufficient)

#### Backend Validation
```python
# In backend API view
def get_collection_for_share(collection_id):
    collection = Collection.objects.get(id=collection_id)
    if not collection.is_public:
        raise PermissionDenied("This collection is not public")
    return collection
```

#### Optional: Simple Password Protection (Future Enhancement)
- Add `share_password` field to Collection model
- If set, require password input before showing content
- Store password hash in backend
- Session-based access after correct password

---

## 📦 Data Requirements

### Backend API Endpoints (Already Available)
```
GET /api/collections/{id}/
  → Returns full collection with all nested data:
    - locations (with images, visits, dates)
    - transportations
    - lodging
    - notes (with markdown content)
    - checklists
    - start_date, end_date
    - is_public flag
```

### Additional Data Needs
None! The existing API provides everything needed.

### Frontend Data Transformations

#### Day Calculation
```typescript
// Calculate total days in trip
const startDate = new Date(collection.start_date)
const endDate = new Date(collection.end_date)
const totalDays = Math.ceil((endDate - startDate) / (1000 * 60 * 60 * 24)) + 1

// Get all items for a specific day
const dayItems = {
  locations: groupLocationsByDate(collection.locations, startDate, totalDays)[dayIndex],
  transportations: groupTransportationsByDate(collection.transportations, startDate, totalDays)[dayIndex],
  notes: groupNotesByDate(collection.notes, startDate, totalDays)[dayIndex],
  // ... etc
}
```

#### Last Viewed Day (localStorage)
```typescript
// Store: When user views a day
localStorage.setItem(`adventurelog_share_${collectionId}_lastDay`, dayNumber)

// Retrieve: On landing page
const lastViewedDay = localStorage.getItem(`adventurelog_share_${collectionId}_lastDay`) || 1

// Show "new content" badge if days were added since last visit
const lastKnownTotalDays = localStorage.getItem(`adventurelog_share_${collectionId}_totalDays`)
const hasNewDays = totalDays > parseInt(lastKnownTotalDays || '0')
```

---

## 🛠️ Technical Implementation Plan

### Phase 0: Backend Extension (Days 1-2)
**Goal:** Add `order` field to enable manual content sequencing

#### Tasks:
1. **Add order field to models**
   - Update `Note`, `Visit`, `Transportation`, `Lodging`, `Checklist` models
   - Add `order = IntegerField(null=True, blank=True, default=None)`
   - Create and run migrations

2. **Update serializers**
   - Add `order` field to all content serializers
   - Ensure API returns order field

3. **Add batch update endpoint**
   - Create `PATCH /api/collections/{id}/reorder/` endpoint
   - Accept array of `{type, id, order}` objects
   - Update order for multiple items at once

4. **Test backend changes**
   - Verify migrations work
   - Test order field in API responses
   - Test batch update endpoint

**Deliverable:** Backend supports order field, API returns it

---

### Phase 1: Core Blog View Foundation (Days 3-5)
**Goal:** Create basic shareable blog view with post-based display

#### Tasks:
1. **Create new route structure**
   - `src/routes/share/[id]/+page.svelte` - Landing page
   - `src/routes/share/[id]/+page.server.ts` - Data loading
   - `src/routes/share/[id]/post/[index]/+page.svelte` - Post view
   - `src/routes/share/[id]/post/[index]/+page.server.ts` - Post data loading

2. **Build core components**
   - `ShareLayout.svelte` - Wrapper with header/footer
   - `ShareHeader.svelte` - Navigation bar
   - `DayView.svelte` - Main day content renderer
   - `HeroGallery.svelte` - Full-width image carousel

3. **Implement data fetching**
   - Server-side collection fetch in `+page.server.ts`
   - Check `is_public` flag, return 403 if private
   - Pass data to components via `$page.data`

4. **Basic styling**
   - Use Tailwind CSS (already in project)
   - Use DaisyUI for consistency
   - Mobile-first responsive design
   - Clean, minimal aesthetic

**Deliverable:** Can view a single day in blog format at `/share/[id]/day/1`

---

### Phase 2: Navigation & Pagination (Week 1-2)
**Goal:** Add full navigation system with last-viewed memory

#### Tasks:
1. **Day navigation component**
   - `DayNavigator.svelte`
   - Prev/Next buttons
   - Day dropdown selector
   - Progress bar showing position in trip
   - Total days counter

2. **Landing page with overview**
   - Hero section with trip summary
   - Week-based quick navigation
   - "Continue Reading" button (goes to last viewed day)
   - Stats: total days, countries, distance, etc.

3. **localStorage integration**
   - Track last viewed day
   - Track known total days (for "new content" detection)
   - Show badge when new days added

4. **URL state management**
   - Browser back/forward works correctly
   - Share-specific day URLs: `/share/{id}/day/4`
   - Scroll position preservation

**Deliverable:** Full navigation system with memory

---

### Phase 3: Rich Content Display (Week 2)
**Goal:** Beautiful presentation of photos, notes, maps

#### Tasks:
1. **Photo gallery system**
   - `PhotoGallery.svelte` component
   - Grid layout (3 cols desktop, 2 cols tablet, 1 col mobile)
   - Lightbox modal for full-screen viewing
   - Image lazy loading for performance
   - Primary image selection (hero vs. gallery)

2. **Story content renderer**
   - `StoryContent.svelte`
   - Markdown rendering with proper styling
   - Typography enhancements (headings, spacing)
   - Inline image support
   - Link styling

3. **Location story cards**
   - `LocationStoryCard.svelte`
   - Expanded format (not compact tile)
   - Inline details: address, rating, category
   - Image carousel integrated
   - Visit information if multiple visits

4. **Map integration**
   - `RouteMap.svelte`
   - Embed MapLibre map for each day
   - Show locations visited that day
   - Optional route line between locations
   - Click marker to see location details

5. **Transportation display**
   - `TransportationStory.svelte`
   - Narrative format: "Journey from X to Y"
   - Distance, duration, dates
   - Transportation type with icon/emoji

**Deliverable:** Rich, visually appealing day view with all content types

---

### Phase 4: Polish & Enhancements (Week 2-3)
**Goal:** Add finishing touches and optional features

#### Tasks:
1. **SEO & metadata**
   - `<meta>` tags with `noindex, nofollow`
   - Open Graph tags for link previews
   - Dynamic title/description per day
   - Favicon and branding

2. **Performance optimization**
   - Image optimization (WebP format, srcset)
   - Lazy loading images below fold
   - Code splitting for routes
   - Minimize bundle size

3. **Accessibility**
   - ARIA labels for navigation
   - Keyboard navigation support
   - Alt text for images (from backend data)
   - Focus management for modals

4. **Print styles** (optional)
   - CSS print stylesheet
   - Clean single-page print format
   - Remove navigation for printing

5. **Add "Share as Story" button to existing UI**
   - In `routes/collections/[id]/+page.svelte`
   - Button in header next to other actions
   - Copy share URL to clipboard
   - Toast notification on copy

6. **Loading states & errors**
   - Skeleton loaders while fetching
   - 404 page for invalid collection IDs
   - 403 page for private collections
   - Offline detection and message

**Deliverable:** Production-ready blog view

---

### Phase 5: Drag-and-Drop Ordering UI (Optional - Week 3-4)
**Goal:** Add visual interface for manually ordering content

#### Tasks:
1. **Create ordering modal in collection view**
   - Add "Reorder for Blog" button in existing collection page header
   - Modal shows all collection content in list format
   - Group by date initially, show current order values

2. **Implement drag-and-drop**
   - Use library like `@dnd-kit/core` or `svelte-dnd-action`
   - Drag handles on each item
   - Visual feedback during drag (ghost, placeholder)
   - Auto-scroll when dragging to edges

3. **Order management logic**
   - Calculate new order values on drop (use gaps: 10, 20, 30...)
   - Batch update API call on "Save"
   - Optimistic UI updates
   - Undo/reset functionality

4. **Visual indicators**
   - Show items with explicit order (numbered badges)
   - Show items without order (date-based)
   - Highlight "post boundaries" (first Note = new post)
   - Preview mode: "View as Blog" button

5. **Mobile-friendly ordering**
   - Touch support for drag-and-drop
   - Alternative: Up/Down buttons for mobile
   - Numbered list view as fallback

**Deliverable:** Full drag-and-drop interface for content ordering

**Example UI:**
```
┌────────────────────────────────────────┐
│  Reorder Content for Blog View         │
│  [Preview Blog] [Reset] [Save]         │
├────────────────────────────────────────┤
│  ☰ [1] Note: "Arrival in Granada"     │ ← Drag handle
│     Jan 4, 2026                        │
│                                        │
│  ☰ [2] Transport: Car (NL → Granada)  │
│     Jan 2-4, 2026                      │
│                                        │
│  ☰ [3] Note: "Exploring Albaicín"     │
│     Jan 4, 2026                        │
│                                        │
│  ☰ [4] Location: Albaicín              │
│     Visited Jan 4, 2026                │
│                                        │
│  ☰ [5] Note: "The Alhambra"            │ ← New post starts here
│     Jan 5, 2026 [NEW POST BOUNDARY]   │
│                                        │
│  ☰ [6] Location: Alhambra              │
│     Visited Jan 5, 2026                │
│                                        │
│  📅 Lodging: Hotel (no order)          │ ← No drag handle (auto-ordered)
│     Jan 4-7, 2026                      │
└────────────────────────────────────────┘
```

---

### Phase 6: Future Enhancements (Optional)
**Goal:** Advanced features for enhanced storytelling

#### Potential Additions:
1. **Comments system**
   - Allow family/friends to leave comments on days
   - Backend: new Comment model linked to collections
   - Frontend: comment thread at bottom of each day
   - Notification to creator on new comments

2. **Password protection**
   - Add `share_password` field to Collection
   - Password input modal on share page
   - Session-based access after correct password

3. **Print-to-PDF**
   - Generate PDF of entire trip
   - Server-side rendering (Puppeteer/Playwright)
   - Download button on landing page

4. **Social sharing**
   - Share buttons for Facebook, WhatsApp, email
   - Custom share text per day
   - Preview card customization

5. **Alternative views**
   - Map-centric view: timeline on map
   - Calendar view: month calendar with day links
   - Photo grid view: all photos from trip

6. **Analytics for creator**
   - View count per day
   - Which days are most popular
   - Visitor locations (anonymous)
   - Backend integration with Umami (already configured)

7. **Export options**
   - Export to Markdown/HTML
   - Export images in zip
   - Export GPX tracks

---

## 🎨 Styling Guidelines

### Color Scheme
- Use existing DaisyUI theme system
- Blog view should be clean and minimal
- Focus on content, not chrome
- High contrast for readability
- Respect user's light/dark mode preference

### Typography
```css
/* Headers */
.day-title { @apply text-4xl md:text-5xl font-bold mb-2; }
.day-subtitle { @apply text-xl md:text-2xl text-base-content/70; }

/* Story content */
.story-content { @apply prose lg:prose-xl max-w-none; }
.story-content p { @apply mb-4 leading-relaxed; }
.story-content h2 { @apply text-3xl font-semibold mt-8 mb-4; }

/* Navigation */
.day-nav { @apply text-sm font-medium tracking-wide uppercase; }
```

### Spacing
- Generous white space around content
- Max content width: 1200px (wider for galleries)
- Consistent padding: 1rem mobile, 2rem desktop
- Section spacing: 3-4rem between major sections

### Images
- Full-width hero: 100vw, max-height: 70vh
- Gallery images: aspect-ratio 4:3 or 3:2
- Image captions: small text below, subtle color
- Hover effects: subtle scale or opacity change

---

## 🚀 Deployment Strategy

### Local Development
```bash
# Frontend (SvelteKit)
cd adventurelog-app/frontend
npm install
npm run dev
# → http://localhost:5173

# Backend (Django)
cd adventurelog-app/backend
python manage.py runserver
# → http://localhost:8000
```

### Production Deployment (Docker)
No changes needed to Docker setup. The new share routes are part of the same frontend app.

```bash
# On production server
docker compose up -d

# Frontend will automatically include new routes
# Share links will be: https://your-domain.com/share/{collection-id}
```

### Environment Variables
No new variables needed. Existing `.env` configuration supports everything.

---

## 🧪 Testing Plan

### Manual Testing Checklist
- [ ] Create test collection with 7 days of content
- [ ] Mark collection as public
- [ ] Access share URL without login
- [ ] Verify all day views render correctly
- [ ] Test navigation: prev/next, dropdown, landing page
- [ ] Test localStorage: last viewed day persists
- [ ] Test "new content" badge when days added
- [ ] Test on mobile: responsive, swipe gestures
- [ ] Test on tablet: layout adjusts properly
- [ ] Test in multiple browsers: Chrome, Firefox, Safari
- [ ] Test with long notes: markdown rendering
- [ ] Test with many photos: gallery performance
- [ ] Test with no photos: graceful fallback
- [ ] Verify robots meta tags present
- [ ] Test 404 for invalid collection ID
- [ ] Test 403 for private collection

### Edge Cases to Handle
- Collection with no start_date/end_date
- Day with no content (no locations, notes, etc.)
- Day with only transportation (no locations)
- Images that fail to load (broken URLs)
- Very long trip (100+ days) - pagination essential
- Very short trip (1 day) - navigation still makes sense
- Collection updated while viewer is reading
- Multiple notes on same day
- Multiple visits to same location on different days

---

## 📁 File Structure

```
adventurelog-app/frontend/src/
├── routes/
│   ├── share/
│   │   └── [id]/
│   │       ├── +page.svelte                 # Landing page
│   │       ├── +page.server.ts              # Data loader
│   │       ├── +layout.svelte               # Share layout wrapper
│   │       ├── day/
│   │       │   └── [number]/
│   │       │       ├── +page.svelte         # Day view
│   │       │       └── +page.server.ts      # Day data loader
│   │       └── map/
│   │           ├── +page.svelte             # Full trip map
│   │           └── +page.server.ts          # Map data loader
│   └── collections/
│       └── [id]/
│           └── +page.svelte                 # (modify to add share button)
│
├── lib/
│   ├── components/
│   │   ├── share/                           # New: Share-specific components
│   │   │   ├── ShareHeader.svelte
│   │   │   ├── ShareFooter.svelte
│   │   │   ├── DayNavigator.svelte
│   │   │   ├── DayView.svelte
│   │   │   ├── HeroGallery.svelte
│   │   │   ├── PhotoGallery.svelte
│   │   │   ├── StoryContent.svelte
│   │   │   ├── LocationStoryCard.svelte
│   │   │   ├── RouteMap.svelte
│   │   │   └── TransportationStory.svelte
│   │   ├── LocationCard.svelte              # Existing (may reuse parts)
│   │   ├── CardCarousel.svelte              # Existing (may reuse)
│   │   └── ...
│   │
│   ├── utils/
│   │   └── shareUtils.ts                    # New: Share-specific utilities
│   │       ├── getLastViewedDay()
│   │       ├── setLastViewedDay()
│   │       ├── hasNewContent()
│   │       └── generateShareUrl()
│   │
│   ├── index.ts                             # Existing (use grouping functions)
│   └── types.ts                             # Existing (use Collection type)
│
└── BLOG_REDESIGN_PLAN.md                     # This file
```

---

## 🔄 Git Workflow for Local Development → Production

### Recommended Approach: Fork the Repository

Since AdventureLog is an open-source project, the cleanest approach is:

#### Step 1: Fork on GitHub
1. Go to https://github.com/seanmorley15/AdventureLog
2. Click "Fork" button (top right)
3. Creates: `https://github.com/YOUR_USERNAME/AdventureLog`

#### Step 2: Add Your Fork as Remote
```bash
cd C:\Code\AdventureLog\adventurelog-app

# Check current remotes
git remote -v
# origin  https://github.com/seanmorley15/AdventureLog.git (fetch)
# origin  https://github.com/seanmorley15/AdventureLog.git (push)

# Rename original remote to 'upstream'
git remote rename origin upstream

# Add your fork as 'origin'
git remote add origin https://github.com/YOUR_USERNAME/AdventureLog.git

# Verify
git remote -v
# origin    https://github.com/YOUR_USERNAME/AdventureLog.git (fetch)
# origin    https://github.com/YOUR_USERNAME/AdventureLog.git (push)
# upstream  https://github.com/seanmorley15/AdventureLog.git (fetch)
# upstream  https://github.com/seanmorley15/AdventureLog.git (push)
```

#### Step 3: Create Feature Branch
```bash
# Create branch for blog redesign
git checkout -b feature/blog-view

# Make your changes, commit frequently
git add .
git commit -m "Add share route structure"

# Push to your fork
git push origin feature/blog-view
```

#### Step 4: Deploy to Production
```bash
# On your production server
cd /path/to/adventurelog

# Add your fork as remote (if not already)
git remote add mycustomization https://github.com/YOUR_USERNAME/AdventureLog.git

# Fetch and checkout your feature branch
git fetch mycustomization
git checkout feature/blog-view

# Pull latest changes
git pull mycustomization feature/blog-view

# Rebuild and restart
docker compose down
docker compose build
docker compose up -d
```

#### Step 5: Sync with Upstream (Get Official Updates)
```bash
# Periodically pull updates from original repo
git fetch upstream
git checkout main
git merge upstream/main

# Rebase your feature branch on latest main
git checkout feature/blog-view
git rebase main

# Resolve any conflicts, then push
git push origin feature/blog-view --force-with-lease
```

### Alternative: Direct Branch on Cloned Repo

If you don't want to fork:

```bash
# Just create a branch on the cloned repo
git checkout -b custom/blog-view

# Commit changes
git add .
git commit -m "Add blog view"

# On production server, pull this branch
# (requires push access to a remote you control)
```

### Recommended Git Strategy Summary

```
Upstream (Original)          Your Fork                    Local Dev              Production Server
┌─────────────────┐         ┌─────────────────┐         ┌──────────────┐       ┌──────────────┐
│ seanmorley15/   │         │ YOUR_USERNAME/  │         │ C:\Code\...  │       │ /var/www/... │
│ AdventureLog    │────────>│ AdventureLog    │<───────>│              │       │              │
│                 │  fork   │                 │ push    │ feature/     │       │ feature/     │
│ main branch     │         │ main branch     │ pull    │ blog-view    │──────>│ blog-view    │
│                 │         │ feature/        │         │              │ pull  │              │
│ (official)      │         │ blog-view       │         │ (develop)    │       │ (deploy)     │
└─────────────────┘         └─────────────────┘         └──────────────┘       └──────────────┘
        │                            │
        │ pull updates               │
        └────────────────────────────┘
         (stay in sync)
```

---

## 📋 Success Criteria

The blog redesign is complete when:

✅ Can access collection via `/share/{id}` without authentication
✅ Landing page shows trip overview with hero image and stats
✅ Can navigate day-by-day with prev/next buttons
✅ Day dropdown shows all days, jumps to selected day
✅ Progress bar shows position in trip
✅ Last viewed day is remembered in browser
✅ "New content" badge appears when days are added
✅ Full-width hero images display for each day
✅ Photo galleries show all images in grid, expandable to lightbox
✅ Notes render as markdown with proper styling
✅ Locations show inline with details (address, rating, etc.)
✅ Transportation shows journey narrative
✅ Embedded map shows day's locations
✅ Mobile responsive (works on phones and tablets)
✅ Robots meta tags prevent search engine indexing
✅ Private collections return 403 error
✅ Invalid collection IDs return 404 error
✅ "Share as Story" button added to existing collection page
✅ Share URL copies to clipboard
✅ Works in production Docker deployment

---

## 📞 Questions & Decisions Log

### Decision 1: Pagination Strategy
**Question:** How to handle 14-week (98-day) trip?
**Decision:** Day-by-day pagination with smart navigation
**Rationale:** One long page = poor performance and overwhelming. Individual days allow focused reading and easy sharing of specific days.

### Decision 2: Privacy Model
**Question:** How to share without being public?
**Decision:** Secret URLs (UUID-based) with `noindex` meta tags
**Rationale:** Balances ease of sharing with privacy. No password needed but not discoverable by search engines.

### Decision 3: Navigation Memory
**Question:** How to help viewer return to where they left off?
**Decision:** localStorage tracking with "new content" badges
**Rationale:** Better UX for long trips. Viewers can read at own pace and return later.

### Decision 4: Photo Display
**Question:** How to show photos?
**Decision:** Full-width hero + grid gallery with lightbox
**Rationale:** Matches Polarsteps aesthetic, prioritizes visual storytelling.

### Decision 5: Existing UI
**Question:** Modify or add parallel view?
**Decision:** Add parallel view, keep existing UI intact
**Rationale:** Preserves data management interface, clean separation of concerns.

---

## 🎓 Learning Resources

### SvelteKit Documentation
- Routing: https://kit.svelte.dev/docs/routing
- Loading data: https://kit.svelte.dev/docs/load
- Layouts: https://kit.svelte.dev/docs/layouts

### DaisyUI Components
- Documentation: https://daisyui.com/components/
- Themes: https://daisyui.com/docs/themes/

### Markdown Rendering
- Library: https://www.npmjs.com/package/marked
- Or: https://www.npmjs.com/package/markdown-it

### Image Lightbox
- Options: photoswipe, yet-another-react-lightbox, svelte-lightbox
- Or build custom with Svelte transitions

### LocalStorage
- MDN: https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage
- Svelte stores: https://svelte.dev/docs/svelte-store

---

## 📅 Timeline Estimate

**Total Duration:** 2-3 weeks (part-time development)

- **Week 1 (10-15 hours)**
  - Phase 1: Core blog view foundation (5-7 hours)
  - Phase 2: Navigation & pagination (5-8 hours)

- **Week 2 (12-18 hours)**
  - Phase 3: Rich content display (8-12 hours)
  - Phase 4: Polish & enhancements (4-6 hours)

- **Week 3 (5-8 hours)**
  - Testing & bug fixes (3-5 hours)
  - Production deployment (2-3 hours)

**Note:** Timeline assumes familiarity with SvelteKit and Tailwind. Adjust if learning curve needed.

---

## 🎯 Next Steps

1. **Review this plan** - Ensure alignment with vision
2. **Set up git workflow** - Fork repo and configure remotes
3. **Create feature branch** - `git checkout -b feature/blog-view`
4. **Start Phase 1** - Build basic share route structure
5. **Iterate and test** - Frequent commits, test on real data
6. **Deploy to production** - When ready, pull to server

---

**Plan Status:** ✅ APPROVED - Ready for Implementation
**Last Updated:** 2026-01-06
**Next Review:** After Phase 1 completion
