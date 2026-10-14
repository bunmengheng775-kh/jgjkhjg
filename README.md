# Study Hub

## Run on your PC (no npm commands)
Install and start Docker Desktop. In PowerShell, go to this project folder and build the app:

    docker build -t study-hub .

Set the initial Owner password securely, then start the server with a persistent data volume:

    $securePassword = Read-Host "Choose an initial Owner password" -AsSecureString
    $env:INITIAL_OWNER_PASSWORD = (New-Object System.Net.NetworkCredential('', $securePassword)).Password
    docker run -d --name study-hub -p 3000:3000 -v study-hub-data:/data -e "INITIAL_OWNER_PASSWORD=$env:INITIAL_OWNER_PASSWORD" study-hub
    Remove-Item Env:INITIAL_OWNER_PASSWORD
    Remove-Variable securePassword

Open `http://localhost:3000` on your PC. The first account is the Owner (default username
`heng`); passwords are stored as hashes. Keep the `study-hub-data` Docker volume to preserve
accounts, file records, and uploads when updating or recreating the container.
Opening `public/index.html` or using VS Code Live Server can display the page, but does not run
the Study Hub server. On a local file or private Wi-Fi address, if the server is unavailable,
login falls back to a browser-only demo. Demo accounts and roles are not verified, passwords
are not saved, and data is not shared with other devices. The online HTTPS app does not use
this demo fallback.

## Accounts and permissions
Choose the role assigned to your account on the login screen. Sign in as `heng` (Owner) and use
**Manage users** to create accounts, change access, or delete accounts. The selected role must
match the account; choosing Owner does not grant Owner permissions.
Students can view materials; Teachers can upload, rename, caption, and delete materials.
Only the Owner can manage accounts. The owner role is reserved for the existing `heng` account,
which cannot be deleted in the user manager.

On the server, account records and password hashes are saved in `DATA_DIR/data/users.json`;
passwords are never stored as plain text. With the Render Blueprint, `DATA_DIR` is `/var/data`
on the persistent disk. The browser-only demo does not save or verify accounts or passwords
and cannot share them with other devices.

## Open it on your phone
Connect your phone and PC to the same Wi-Fi network. On your PC, run `ipconfig` and find its
IPv4 address, then open `http://<PC-IPv4-address>:3000` in your phone's browser (for example,
`http://192.168.1.23:3000`). Do not use `localhost` on your phone; that points to the phone
itself. If Windows Firewall asks, allow Docker Desktop on your private network.

Tap **Add to Home Screen** in Study Hub to install the shortcut. On iPhone/iPad, use Safari's
Share button and choose **Add to Home Screen**. The app shell can open offline, but signing in
and loading materials still requires a connection to the PC.

## Share it online
For access from outside your home Wi-Fi, deploy the server online. The included Render Blueprint
uses a persistent disk, so user accounts, uploaded files, and file records are shared between
devices and survive deploys. Render installs dependencies during deployment; you do not need to
run `npm install` on your PC.

1. Push this project to a GitHub repository.
2. In Render, choose **New > Blueprint** and select that repository; Render reads `render.yaml`.
3. Set `INITIAL_OWNER_PASSWORD` privately in the service's Environment settings before its first
   deploy. Keep the password out of Git.
4. After Render creates the service, set `PUBLIC_URL` to its HTTPS address and redeploy.
5. Open that same HTTPS address on every phone or PC. Teachers' uploads appear in other users'
   materials list automatically within about eight seconds.

The Blueprint uses a paid Render plan because persistent disks require one. Do not remove or
replace the disk after users have started uploading files.

## Deploy backend to Cloudflare Workers
The Worker website alone serves only the app page. The included `worker.js` adds the API and
saves accounts, exercises, and answers in D1, with uploaded files in R2. To make a Cloudflare
Worker deployment share data between devices:

1. In Cloudflare, create a D1 database named `study-hub` and an R2 bucket named
   `study-hub-uploads`.
2. Open the D1 database's SQL Console and run the SQL in `migrations/0001_initial.sql`.
3. Copy the D1 database ID into `wrangler.jsonc`, replacing `REPLACE_WITH_D1_DATABASE_ID`.
4. Commit/push the updated project to GitHub, then connect that repository to the Worker build.
   Set the deploy command to `npx wrangler@4 deploy` and use `wrangler.jsonc` as the Worker
   configuration. Cloudflare deploys `worker.js` and the `public` assets together; npm is not
   needed on your PC.
5. In the Worker settings, add `INITIAL_OWNER_PASSWORD` as a secret (8–128 characters). Add a
   long, random `API_KEY` secret if you use the public-files integration. Do not put secrets in
   Git.
6. Redeploy and check `https://YOUR-WORKER.workers.dev/api/health`; it should return
   `{"ok":true}`. Then open your Worker URL on each device. The first login initializes the
   Owner account (`heng`). Create teacher and student accounts from the Owner account.

The Worker accepts uploads up to 20 MB per file and 30 MB total per upload. The current hosted
Worker must be redeployed with this backend; until then its `/api/...` routes return 404 and it
cannot save or share accounts, uploads, exercises, or submissions.

Teachers can create a worksheet with instructions and written-answer questions. Students submit
answers in Study Hub, and teachers can review and see submissions update while signed in. Teachers
can also upload files and photos as course materials.

## Folders
- public/css/style.css   all styling
- public/js/api.js       calls to the server
- public/js/app.js       screens (login, list, viewer)
- data/users.json        local persistent user accounts (excluded from Git)
- data/files.json        uploaded file records
- data/exercises.json    worksheet questions
- data/submissions.json  student exercise answers
- uploads/               the uploaded files
- server.js              login + upload + public API + webhook

## Let other drives / apps see uploads
1. Pull list (any app):
   curl -H "x-api-key: YOUR_KEY" http://localhost:3000/api/public/files
   Each item has a `url` to download the file.
2. Push (instant): set WEBHOOK_URL. On every upload/rename/delete the server POSTs
   {"event":"uploaded|updated|deleted","file":{id,name,caption,kind,size,url},"at":"..."}
   Connect it to Zapier / Make / n8n / Google Apps Script -> Google Drive, OneDrive, Dropbox "Upload file from URL".
