# Foreman Sign-In

Mobile-first daily attendance app for the Webcor Concrete foreman crew at the LA Convention Center project. Replaces the paper "Subcontractor Daily Employee Sign-In Sheet" with a phone-friendly form, signature capture, and a one-tap PDF export that matches the original form.

## Live

- App: https://dyap123.github.io/foreman-attendance/

## How it works

- **Foreman flow**: Open the app, tap your name (manager-controlled list), sign in for today (or backfill up to 7 days), draw your signature, submit. View your history and weekly hours.
- **Manager flow**: Tap "Manager" tab, enter PIN (default `8888`, change it in Config), manage the roster, edit any historical entry, and export a weekly PDF that mirrors the paper form layout. Share via the Apple share sheet (AirDrop, Mail, Google Drive, etc.).

## Stack

- Single-file static SPA — `index.html`. No build step.
- Firebase Realtime Database (shared with CUP Pour Dashboard, namespace `foreman-attendance/`).
- SignaturePad for signature capture.
- jsPDF + jsPDF-AutoTable for PDF export.
- `navigator.share` for native share sheet on iOS/Safari.

## Local dev

```sh
cd ~/foreman-attendance
python3 -m http.server 8484
# Open http://localhost:8484
```

Note: `navigator.share` requires HTTPS, so local testing falls back to file download. The deployed site (GitHub Pages = HTTPS) will trigger the native share sheet on iPhone.

## Deploy

```sh
cd ~/foreman-attendance
git init
gh repo create dyap123/foreman-attendance --public --source=. --remote=origin
git add . && git commit -m "Initial commit"
git push -u origin main
gh api -X POST /repos/dyap123/foreman-attendance/pages -f source.branch=main -f source.path=/
```

## Data model (Firebase RTDB, namespace `foreman-attendance/`)

```
config/
  primeContractor, projectName, subcontractor, managerPin
foremen/{id}/
  name, classification, active, order
attendance/{YYYY-MM-DD}/{foremanId}/
  timeIn, timeOut, totalHours, injured,
  signatureType ("drawn" | "typed" | "off"),
  signatureDataUrl, signatureText,
  signedAt, signedBy ("self" | "manager")
```

## Default manager PIN

`8888` — change it in the Manager → Project Config tab on first use.
