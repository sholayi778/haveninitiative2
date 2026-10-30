# Haven Initiative website

The website of **Haven Initiative for Women and Children Development and Safety**, Ibadan, Nigeria. It is a frontend-only rebuild of haveninitiative.org.

The original site's audit, content inventory and every correction or exclusion made during the rebuild are documented in [`docs/AUDIT.md`](docs/AUDIT.md).

## 1. Overview

The site has 10 routes:

| Route | Page |
|---|---|
| `/` | Home |
| `/about` | About, values and Board of Trustees (`#board`) |
| `/programme` | Programme activities and thematic areas |
| `/gallery` | Photo albums with a lightbox |
| `/gallery/share` | Photo sharing (upload interface, not in the main nav, `noindex`) |
| `/events` | Upcoming and past events |
| `/blog`, `/blog/:slug` | News and articles |
| `/contact` | Contact details and enquiry form |
| `/volunteer` | Volunteer registration |
| anything else | 404 page |

Old WordPress URLs (`/portfolio`, `/member/...`) redirect to their new equivalents.

## 2. Technology

| | |
|---|---|
| Framework | React 19 + TypeScript, built with Vite 8 |
| Styling | Tailwind CSS 4. Design tokens are in `src/styles/index.css`; colours are sampled from the logo (green #039254, orange #e58b3c). |
| Routing | React Router 8 (data router) |
| Icons | Lucide React; brand marks drawn in `src/components/BrandIcons.tsx` |
| Fonts | Bricolage Grotesque (display) and Atkinson Hyperlegible (body), self-hosted via Fontsource. No third-party font requests. |
| Hosting | Vercel (`vercel.json`) |

There is no animation library, state library, backend or database.

## 3. Directory structure

```
docs/AUDIT.md                  audit of the old site, inventory, gaps
public/
  images/                      migrated original photographs (filled by the script)
  favicon/site.webmanifest
  robots.txt, sitemap.xml
scripts/migrate-images.mjs     downloads original images into public/images
src/
  components/                  Header, Footer, SiteImage, Lightbox, GalleryGrid, TrusteeShowcase, forms…
  data/                        ALL editable content (see section 8)
    images.manifest.json       every original image: source URL, page, local path, alt text
  features/uploads/            image-upload feature (port, adapters, hook, component)
  hooks/                       usePageMeta (titles/OG tags), useSubmission (forms)
  layouts/SiteLayout.tsx
  lib/                         image lookup, dates, form submission, collections
  pages/                       one file per route
  styles/index.css             design tokens and base styles
  types/content.ts             content types
vercel.json                    SPA rewrites, redirects, cache headers
```

## 4. Local setup

You need Node.js 20.19 or newer.

```bash
npm install
npm run images:migrate     # download the original photos into public/images
```

## 5. Development server

```bash
npm run dev                # http://localhost:5173
```

If an image has not been migrated yet, the site loads it from haveninitiative.org automatically. If that also fails, it shows a labelled "Image unavailable" panel instead of a broken image.

## 6. Production build

```bash
npm run build              # runs the image migration first, then type-checks and builds to dist/
npm run preview            # serve dist/ locally
```

`npm run images:check` lists which images are present locally without downloading anything.

## 7. Deploy to Vercel

1. The original photos are already included in `public/images/` (34 of 35; see section 12), so they deploy with the site and don't depend on the old server.
2. Push the project to a GitHub repository.
3. In Vercel, go to **Add New → Project**, import the repository, and keep the detected settings (framework Vite, build `npm run build`, output `dist`).
4. Optionally add the environment variables from `.env.example` (see sections 10 and 12).
5. Deploy, then point the `haveninitiative.org` domain at the project in **Settings → Domains**.

`vercel.json` rewrites all page routes to `index.html`, so refreshing or deep-linking `/about` or `/blog/some-article` works. If you skip step 1, the `prebuild` step downloads the images during each Vercel build. That only works while the old server is still online.

## 8. Updating content

All text lives in `src/data/`. You don't need to edit components.

| To change… | Edit |
|---|---|
| Phone, email, address, social links, WhatsApp | `site.ts` |
| About text, vision, mission, values, thematic areas, success story | `content.ts` |
| Programme activities (add a `description` once verified) | `programmes.ts` |
| Board of Trustees (names, roles, optional `bio`) | `board.ts` |
| Gallery albums and order | `gallery.ts` |
| Events | `events.ts` |
| Blog articles | `articles.ts` |

**Events.** Give an event an ISO `date` such as `'2026-12-05'`:

- If the date is today or later (Lagos time), it appears under **Upcoming** and on the homepage.
- Once the date passes, it moves to **Past activities** automatically.
- An event with no `date` is never shown as upcoming.
- An optional `link` adds a registration or details button.

**Articles.** Each article needs `slug`, `title`, `date`, `excerpt` and `body`, an array of paragraphs where a paragraph starting with `## ` becomes a subheading. There is a commented example in `articles.ts`.

- Articles are sorted by date automatically.
- Each one gets its own page at `/blog/<slug>` with previous/next links and related articles.
- Once at least one article exists, the homepage's "News and events" section switches from the "Follow our work" strip to article previews.

After adding articles or pages, also add their URLs to `public/sitemap.xml`.

## 9. Adding or replacing images

1. Put the file in `public/images/` (for example `public/images/gallery/childrens-day/childrens-day-5.jpg`).
2. Add an entry to `src/data/images.manifest.json` with a unique `id`, the `file` path relative to `public/images`, and descriptive `alt` text. Set `"used": true`, add `"album": "childrens-day"` to place it in a gallery album, and leave `remote` as `""` with `fallbacks` as `[]` for new photos.
3. Reference the image anywhere by its `id`, such as `coverImage`, `images: [...]` or `<SiteImage id="…" />`.

To replace an image, overwrite the file and keep the same name. Keep photos under about 2,000 px on the long edge and 500 KB where possible; the original WhatsApp exports are already in that range.

Alt text for every photo was written after viewing it. When you add photos, describe what each one shows (for example "Volunteers handing out school bags to pupils"), and avoid identifying children by name. Each manifest entry can also set `focus`, a CSS `object-position` such as `'50% 30%'`, which keeps faces in frame when the photo is cropped.

## 10. How the image upload works

`/gallery/share` uses a reusable `ImageUploader` component (`src/features/uploads/`). It supports:

- Choosing files or drag-and-drop, with multiple selection.
- JPG, PNG and WebP up to 8 MB each, 12 at a time. Other files are rejected with a specific message.
- Instant previews, removing individual photos, or clearing all.
- Per-photo status and progress, and retrying failed photos.
- Full keyboard operation and screen-reader announcements.

Rules are passed as a prop: `<ImageUploader rules={{ accept: [...], maxBytes, maxFiles }} />`.

The uploader talks to an `UploadAdapter` interface (`types.ts`), so storage can change without touching the UI.

- **Default: `localPreviewAdapter`.** Nothing leaves the browser. Previews use object URLs, which are released when photos are removed, cleared, or the page closes. The page tells users clearly that photos are not uploaded and will disappear on refresh, and gives them the email address to send photos instead.
- **Cloudinary: `cloudinaryAdapter`.** This is enabled automatically when both Cloudinary variables are set (section 12).

The upload feature does **not** add photos to the website gallery. The gallery is built from `src/data/` at deploy time, and there is no content-management dashboard.

## 11. Making uploads persist (Cloudinary)

1. Create a free Cloudinary account.
2. Go to **Settings → Upload → Upload presets → Add preset**. Set **Signing mode: Unsigned**, a folder (the site sends `haven-initiative/submissions`), allowed formats `jpg, png, webp`, and a maximum file size.
3. In Vercel's environment variables, set `VITE_CLOUDINARY_CLOUD_NAME` and `VITE_CLOUDINARY_UPLOAD_PRESET`, then redeploy.

Notes:

- No API secret is ever placed in the browser.
- Unsigned uploads can be made by anyone who finds the page, so keep the preset restricted. Consider Cloudinary's moderation add-on, or put the page behind Vercel password protection if you want only volunteers to use it.
- Uploaded files are reachable by their Cloudinary URL. They do not appear on this website unless someone adds them to `src/data/`.

To use another provider, write an object that implements `UploadAdapter` and return it from `getUploadAdapter()` in `adapters.ts`.

## 12. Environment variables and remaining integrations

All variables are optional (see `.env.example`).

| Variable | Effect when set | When unset |
|---|---|---|
| `VITE_FORM_ENDPOINT` | Contact and volunteer forms POST JSON to this URL. Works with Formspree, Getform, Basin and similar services. | Forms validate, then say plainly **"Your message has not been sent"** and offer a pre-filled email and the phone number. |
| `VITE_CLOUDINARY_CLOUD_NAME` and `VITE_CLOUDINARY_UPLOAD_PRESET` | Photo uploads go to Cloudinary. | Previews stay in the browser only. |

**Known limitations**

- **Missing portrait:** Mr Kunle Igbaroola's portrait no longer exists on the old server, so the site shows his initials instead. To add it, save the photo as `public/images/board/kunle-igbaroola.jpg` and add `image: 'board-kunle-igbaroola'` to his entry in `src/data/board.ts`.
- **Image resolution:** the original photos are WhatsApp exports, 720 to 1280 px wide. The homepage hero photo (720 × 687 px) may look slightly soft on large high-resolution screens. If the organisation has the original camera files, replace the image files and keep the same names.
- **Search engines:** this is a single-page app.
  - Page titles and descriptions are set in the browser, which Google handles but some social-media previewers do not; they fall back to the site-wide tags in `index.html`.
  - Unknown URLs show the 404 page but return HTTP 200 (marked `noindex`).
  - Pre-rendering would fix both if needed.
- **Not included:** donations or payments (none existed on the old site), a privacy notice (needed because the forms collect personal data; see `docs/AUDIT.md`), and a content-management system.

## Quality checks performed

These checks were run with Playwright and Chromium against the production build, using temporary stand-in images that have since been removed:

- **Pages:** all routes render at 1440 px and 390 px with no horizontal overflow and no console errors. Direct loads of `/about`, `/events`, `/portfolio`, an unknown article and an unknown URL all resolve correctly.
- **Lightbox:** opens from the keyboard, arrow keys and buttons cycle and wrap, Escape closes and returns focus to the photo that opened it.
- **Mobile menu:** opens, closes on Escape, and closes after navigating.
- **Uploader:** rejects invalid types and oversize files with specific messages; removal, clearing and confirming work; all object URLs are revoked.
- **Uploader with Cloudinary configured** (requests intercepted): uploads post the unsigned preset and report progress and success; provider errors show with a retry.
- **Forms:** required-field and format validation work, focus moves to the first error, and `?topic=` and `?programme=` presets apply. Unconnected mode shows the not-sent notice with a correct mailto link; connected mode (mocked) posts JSON, confirms and clears the form.

**Not tested:** real Vercel deployment, real Cloudinary and Formspree accounts, the actual original photographs, and screen readers other than through accessible names and roles.
