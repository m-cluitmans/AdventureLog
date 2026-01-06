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
3. **Paginated navigation** - Handle long trips (14 weeks) with day-by-day pagination
4. **Smart navigation** - Remember viewer's last position, highlight new content
5. **Full-width media** - Hero images and photo galleries for visual storytelling
6. **Secret public URLs** - Shareable links not indexed by search engines
7. **Mobile-friendly** - Responsive design for viewing on all devices

---

## 📐 Architecture Design

### Option Selected: Hybrid Approach (Option 1 + Navigation Enhancements)

#### New Routes Structure
```
/share/[collection_id]              → Blog landing page with trip overview
/share/[collection_id]/day/[number] → Individual day view (paginated)
/share/[collection_id]/map          → Full trip map view
```

#### Component Hierarchy
```
ShareLayout.svelte (wrapper for all blog views)
├── ShareHeader.svelte (trip title, date range, navigation)
├── DayNavigator.svelte (pagination, progress bar, "new content" badges)
├── DayView.svelte (main day content renderer)
│   ├── HeroGallery.svelte (full-width photo display)
│   ├── StoryContent.svelte (markdown notes rendered as blog posts)
│   ├── LocationStoryCard.svelte (inline location with expanded details)
│   ├── RouteMap.svelte (embedded map for the day's route)
│   └── TransportationStory.svelte (journey details in narrative format)
└── ShareFooter.svelte (trip stats, social sharing)
```

---

## 🗺️ User Experience Flow

### For Content Creator (You)
1. Use existing AdventureLog interface to add locations, photos, notes, transportation
2. Mark collection as `is_public = true`
3. Click "Share as Story" button in collection management page
4. Copy secret share link: `https://yourserver.com/share/abc123def456`
5. Send link to family/friends

### For Viewers (Family/Friends)
1. Open shared link → lands on trip overview page
2. See trip summary: hero photo, title, date range, total days
3. Navigate to days via:
   - Day dropdown selector
   - Previous/Next buttons
   - Timeline/progress visualization
4. Browser remembers last viewed day (localStorage)
5. On return visit:
   - Opens last viewed day by default
   - Shows badge if new days were added: "New: Day 15-17"
6. Can view full map of entire trip
7. Can navigate entire story chronologically

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

### Phase 1: Core Blog View Foundation (Week 1)
**Goal:** Create basic shareable blog view with single-day display

#### Tasks:
1. **Create new route structure**
   - `src/routes/share/[id]/+page.svelte` - Landing page
   - `src/routes/share/[id]/+page.server.ts` - Data loading
   - `src/routes/share/[id]/day/[number]/+page.svelte` - Day view
   - `src/routes/share/[id]/day/[number]/+page.server.ts` - Day data loading

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

### Phase 5: Future Enhancements (Optional)
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
