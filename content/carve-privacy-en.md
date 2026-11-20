+++
title = "Carve Privacy Policy"
url = "/carve/privacy/en/"
description = "Privacy policy for Carve (새기다), a Bible handwriting app for iPad."
ShowToc = true
ShowReadingTime = false
ShowBreadCrumbs = false
ShowPostNavLinks = false
searchHidden = true
comments = false
# 글 목록 · RSS 에 넣지 않는다(주소로만 연다 — App Store Connect 의 처리방침 URL)
[build]
  list = "never"
+++

[한국어](/carve/privacy/)

Lee Taekseong (the "developer") operates the iPad Bible handwriting app Carve (새기다, the "app"). This policy explains what information the app processes, why, and how you can control it. The Korean version is the authoritative text; this is a translation.

- First effective: July 7, 2022
- Last updated: November 20, 2026

## 1. Purposes of processing

1. **Inquiries and feedback:** to review and answer bug reports, questions and suggestions you send by email or with "Send Feedback" in the app. Replies are shown in the app.
2. **Stability and quality:** to analyze crashes, errors, memory use and other diagnostics and fix problems.
3. **Usage analytics:** to analyze screen views, app launches, feature use and purchase events to improve features.
4. **Advertising:** to show third-party ads through Google AdMob, measure ad performance, prevent fraud and handle ad consent choices.
5. **Saving and syncing your writing:** to store handwriting, favorites and past records on your device and, if you use iCloud, sync them between devices on the same Apple Account. If you choose, the app creates or restores password-protected backup files.
6. **Group challenges, shared handwriting and shared books:** to let members of a group you create or join exchange progress, handwriting you choose to share, and handwriting you place in a shared book.
7. **Reports of inappropriate content:** to review reports about handwriting shared in a group and take it down when needed.

## 2. Retention

1. Email inquiries are deleted without delay once handled, unless the law requires keeping them longer.
2. **Feedback and handwriting reports sent in the app are deleted one year after they are handled.** They are stored in the developer's iCloud (Apple CloudKit public database) so that only the developer can read them. You cannot edit or delete feedback in the app after sending it. To have it deleted sooner, email the contact in Section 8 with the date and title (or category) and it will be deleted promptly.
3. The list of handwriting taken down after a report (the writer's iCloud user identifier and the verse) is kept while the takedown is in effect and deleted when it is lifted.
4. Writing stored on your device is kept until you delete it. iCloud sync data and backup files you saved elsewhere are separate from the app data on the device; deleting the app does not delete them.
5. Group, shared handwriting and shared book data follow Section 3-2.
6. Firebase Crashlytics crash records and related identifiers are kept for 90 days, after which Google begins deletion from its production and backup systems.
7. Google Analytics keeps event-level data for 2 months and user-level data for 14 months. "Reset on new activity" is on, so the retention period for a user identifier restarts when there is new activity. Expired data is removed by Google's monthly deletion process; this setting does not limit standard aggregated reports.
8. Firebase installation identifiers follow Google's per-service retention policies. Removing a Cloud Messaging installation identifier from production and backup systems can take up to 180 days after a deletion request. Ad-related data is kept and processed under Google's applicable policies.

See also: [Firebase privacy and security](https://firebase.google.com/support/privacy), [Google Analytics data retention](https://support.google.com/analytics/answer/7667196), [Google Privacy Policy](https://policies.google.com/privacy).

## 3. Information processed and third-party services

Besides what the App Store provides, SDKs in the app may collect the following automatically. What is collected can vary with the app version, whether you bought ad removal, your region and your consent choices.

1. **Crash, performance and other diagnostics:** crash stacks, errors, device model, OS and app version, memory use and termination counts. Used for diagnostics through Firebase Crashlytics and Google Analytics and for quality and performance analysis by the ad SDK.
2. **Device and app-install identifiers:** app instance ID, Firebase installation ID, identifier for vendor and similar. Used for analytics, ads and SDK features.
3. **Product interaction and ad data:** app launches, screen views, feature use, ad impressions and interactions. Used for usage analytics and ad delivery and measurement. Analytics never include nicknames, group names, invite codes, feedback content or handwriting images.
4. **Purchase history:** purchase events (product ID, name, price) measured by Google Analytics through StoreKit. The developer does not collect card numbers or bank account details.
5. **Coarse location:** country or region estimated from your IP address. The app does not request location permission for precise GPS location.
6. **Other data:** metadata handled by default by SDKs such as Firebase Messaging, including device model, language, time zone and app identifiers.
7. **Email inquiries:** your email address, message and any attachments. Please do not send passwords or backup passwords.
8. **In-app feedback:** the category, title, message and up to 5 attached photos you send with "Send Feedback", together with your device model, OS version and app version, and an iCloud user identifier that tells senders apart. This identifier is generated by Apple for each app and does not contain your name or email address. Sending feedback requires signing in to iCloud, and you confirm a consent notice before sending. When a reply arrives, you are notified with an Apple push notification.
9. **Handwriting reports:** when you report handwriting shared in a group, the reason and details (up to 200 characters), the reported handwriting image and verse, the writer's nickname and iCloud user identifier, the group name and group identifier, and the reporter's device information are sent to the developer in the same way as feedback. The writer is not told about the report.
10. **Your writing:** handwriting, favorites and past writing are stored on your device and, with iCloud, synced to your Apple CloudKit private database. The app is not built to send handwriting images, stroke data or backup passwords to Firebase or AdMob for analytics or advertising. Material you attach to an inquiry or feedback is treated as inquiry or feedback data.

### Third-party services

- **Google Firebase Crashlytics, Google Analytics, Firebase Messaging/Installations:** error diagnostics, usage analytics and SDK features.
- **Google AdMob and User Messaging Platform:** ad delivery and measurement and ad privacy choices. The AdMob app is linked to the Google Analytics app data stream to share usage metrics with AdMob and analyze ad revenue in Analytics.
- **Apple iCloud/CloudKit, StoreKit and push notifications:** syncing your writing, group sharing (CloudKit sharing), storing feedback, reports and invite codes (CloudKit public database), feedback reply notifications, and in-app purchases.

Information may be sent over the network when the app launches, is used, hits an error or requests ads. Google services may process data on global infrastructure, including in the United States. See the official pages for processing locations and policies:

- [Firebase data collection on Apple platforms](https://firebase.google.com/docs/ios/app-store-data-collection)
- [Google Analytics App Store data disclosure](https://support.google.com/analytics/answer/10285841)
- [AdMob data disclosure](https://developers.google.com/admob/ios/privacy/data-disclosure)
- [Firebase processing locations, retention and deletion](https://firebase.google.com/support/privacy)
- [Apple Privacy Policy](https://www.apple.com/legal/privacy/)

## 3-2. Group challenges, shared handwriting and shared books

Data exchanged in group features is stored in the iCloud (Apple CloudKit shared space) of the person who created the group (the group owner) and received by the members' devices. **The developer cannot access this data.** Only the report and invite code information described below is sent to or visible to the developer.

1. **Group challenges:** the group name, members' nicknames, reading range and schedule, and completed chapters are visible to group members. Reflections, notes and verse cards are not shared. Some of the group owner's Apple Account information may reach members' devices as iCloud sharing metadata, but the app does not display, store or log it. Invite codes are stored in the CloudKit public database, and the group name and owner nickname they contain may be visible in the developer's management console.
2. **Shared handwriting:** handwriting is shared only when the group owner turns on "Share Handwriting" for the group and each member turns on sharing of "My Handwriting" (both are off by default). What is shared: images of verses the handwriting recognizer judged as written, the verse, translation, time written and time shared, and likes (who liked which verse). Free text and photos cannot be posted.
   - You can remove shared handwriting at any time by turning off "My Handwriting", taking down that verse, leaving the group, the owner closing sharing, or ending the group. Screenshots another member already took cannot be deleted.
3. **Shared books (writing together):** for chapters you "Add to Book", the verse handwriting images, verses, translation and time added are stored in the group owner's iCloud (the group's shared space) and visible to group members.
   - Before the book is bound, you can remove them at any time with "Remove from Book", releasing the chapter, or leaving the group.
   - When the owner binds the book, the included handwriting and the nickname of the writer of each chapter are saved as one volume on each member's device. After binding, you can delete it only from your own device, not from other members' devices.
   - Bound books are included in device backups but not in iCloud sync or handwriting backup files.
4. **Reports and moderation:** shared handwriting can be reported and is hidden right away on the reporter's screen. Reported handwriting is sent only to the developer with the information in Section 3, item 9; the developer reviews the report and takes down inappropriate handwriting (the takedown list is kept as in Section 2, item 3). Report data is deleted one year after it is handled.

## 4. Destruction of personal information

When personal information is no longer needed because its retention period has passed or its purpose has been achieved, the developer destroys it without delay, using methods that make electronic records unrecoverable. If other laws require keeping it, it is stored separately.

## 5. Your rights

You may at any time request access to, correction, deletion or suspension of processing of your personal information, in writing or by email, directly or through a legal representative. The developer will act on your request without delay after confirming that it comes from you or your authorized representative. Some rights may be limited as provided by the Personal Information Protection Act of Korea.

## 6. Security

The app processes data on the device using the iOS/iPadOS app sandbox and data protection. Handwriting backup files you export are protected with a password you choose. Firebase uses HTTPS in transit, and data stored with third-party services is subject to those providers' access controls and security policies. CloudKit permissions make in-app feedback and reports readable only by the sender and the developer. Please include only the information needed to look into a problem.

## 7. Automatic collection and your choices

The app may automatically collect the identifiers, usage and diagnostic data in Section 3 through the Firebase and Google ad SDKs.

The app does not currently request App Tracking Transparency permission or access to the advertising identifier (IDFA). In-app analytics using device or app-install identifiers is different from tracking across other companies' apps and websites.

In regions where ad consent is required, a choice screen may appear through Google User Messaging Platform, and where ad privacy options are offered you can review and change them in the app's settings. Once an ad removal purchase is confirmed, the app does not run the ad consent request or start the ad SDK.

Ad privacy options or ad removal do not turn off all Firebase Analytics or Crashlytics collection.

Shared handwriting in groups works only when both the owner and each member turn it on, and can be turned off or taken down at any time as described in Section 3-2.

"Delete All Writing" in the app deletes your writing data. It does not delete analytics or diagnostics already sent to Google, feedback or reports already sent, or bound books saved on other members' devices. Backup files you saved separately must be deleted where you saved them.

## 8. Privacy officer and contact

- Privacy officer: Lee Taekseong
- Contact: [retake_joy@naver.com](mailto:retake_joy@naver.com)

Contact this address for any privacy questions, complaints, requests to exercise your rights, or requests to delete feedback or report data. The developer will respond without delay.

## 9. Changes to this policy

- First effective: July 7, 2022
- October 15, 2026: updated the data collected by the Firebase and Google ad SDKs, purposes, retention and deletion, third-party services and your choices to match the current app.
- November 20, 2026 (app 2.4): added in-app feedback (data collected, deletion one year after handling, iCloud storage, how to request deletion), group challenges, shared handwriting and shared books, and handwriting reports and takedowns. Moved this policy to this page.
