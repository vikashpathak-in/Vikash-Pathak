# Firebase setup for the portfolio upload feature

The portfolio frontend and admin page use the existing Firebase project configured in DigiCafe (project ID: digicafe-ee519). These are **rule additions**, not a replacement for your existing DigiCafe rules. Firebase rules are project-wide. Merge the following match blocks into the existing Firestore and Storage rules, preserving every existing rule block.

## Firestore: add inside the existing service cloud.firestore { match /databases/{database}/documents { ... } block

```javascript
function isPortfolioAdmin() {
  return request.auth != null
    && request.auth.token.email_verified == true
    && request.auth.token.email in [
      'digicafe.admin@gmail.com',
      'vikashpathak.mgmt@gmail.com',
      'admin@digicafe.com',
      'support@digicafe.com'
    ];
}

match /portfolioFiles/{fileId} {
  allow read: if isPortfolioAdmin()
    || (resource.data.status == 'published');
  allow create: if isPortfolioAdmin()
    && request.resource.data.status == 'pending'
    && request.resource.data.ownerUid == request.auth.uid;
  allow update: if isPortfolioAdmin();
  allow delete: if isPortfolioAdmin();
}
```

## Cloud Storage: add inside the existing service firebase.storage { match /b/{bucket}/o { ... } block

```javascript
function isPortfolioAdmin() {
  return request.auth != null
    && request.auth.token.email_verified == true
    && request.auth.token.email in [
      'digicafe.admin@gmail.com',
      'vikashpathak.mgmt@gmail.com',
      'admin@digicafe.com',
      'support@digicafe.com'
    ];
}

match /portfolio-pending/{uid}/{fileName} {
  allow read, write: if isPortfolioAdmin() && request.auth.uid == uid
    && request.resource.size < 15 * 1024 * 1024;
}
match /portfolio-published/{fileId}/{fileName} {
  allow read: if true;
  allow write: if isPortfolioAdmin()
    && request.resource.size < 15 * 1024 * 1024;
}
```

Important: for Storage deletes, request.resource is null. If you want the admin to delete pending files, use this safer block instead of the pending block above:

```javascript
match /portfolio-pending/{uid}/{fileName} {
  allow read: if isPortfolioAdmin() && request.auth.uid == uid;
  allow create, update: if isPortfolioAdmin() && request.auth.uid == uid
    && request.resource.size < 15 * 1024 * 1024;
  allow delete: if isPortfolioAdmin() && request.auth.uid == uid;
}
```

The admin page only signs in; it does not create accounts. Create/verify the admin Firebase Authentication account using one of the four allowlisted emails. Do not put a password or service-account key in GitHub. The public portfolio reads only Firestore records with status = published. Draft objects are stored under portfolio-pending and are not public. Publishing copies the approved file to portfolio-published and then changes the Firestore status. If Storage or Firestore rules are not merged and deployed, upload/publish will fail closed.

## Deployment checklist
1. In Firebase Console, confirm Email/Password sign-in is enabled and create/verify your admin account using an allowlisted email.
2. Merge the Firestore match block into your current Firestore rules without removing DigiCafe rules.
3. Merge both Storage match blocks into your current Storage rules, using the safer pending block above.
4. Publish the updated rules in Firebase Console.
5. Test with a small dummy PDF: upload as draft, confirm it is absent from the public page, publish it, then confirm it appears.
6. Never upload employer/customer bank statements, IDs, passwords, tax returns or other confidential personal data. Use dummy or anonymised samples.
