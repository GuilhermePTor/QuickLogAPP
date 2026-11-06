1. Create a free GitHub account, then a NEW repository (private is fine).
2. Click "uploading an existing file" and drag in EVERYTHING from this folder
   (including the hidden .github folder - if your browser skips it, see note below).
3. Open the repo's "Actions" tab -> "Build APK" -> wait ~4-6 min for the green check.
4. Open the finished run -> Artifacts -> download "QuickLog-apk" (a zip containing app-debug.apk).
5. Send app-debug.apk to your phone, tap it, allow "install unknown apps" when asked.
Note: if .github doesn't upload, use Add file -> Create new file, type
.github/workflows/build.yml and paste the contents of that file.
