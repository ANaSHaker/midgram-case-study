<p align="center"><img src="assets/banner.png" alt="MidGram" width="100%"></p>
<h1 align="center">MidGram</h1>
<p align="center"><b>A realtime B2B network for pharmacies and medical representatives: product posts, orders, chat and visit appointments in one Arabic-first app.</b></p>
<p align="center">
  <img src="https://img.shields.io/badge/Platform-Android%20%7C%20iOS-2196F3?style=flat-square" alt="Platform">
  <img src="https://img.shields.io/badge/Flutter-3.x-02569B?style=flat-square&logo=flutter&logoColor=white" alt="Flutter">
  <img src="https://img.shields.io/badge/State-GetX-8A2BE2?style=flat-square" alt="GetX">
  <img src="https://img.shields.io/badge/Backend-Firebase-FFCA28?style=flat-square&logo=firebase&logoColor=black" alt="Firebase">
  <img src="https://img.shields.io/badge/Realtime-Cloud%20Firestore-F57C00?style=flat-square" alt="Firestore">
  <img src="https://img.shields.io/badge/UI-Arabic%20RTL-E0A84A?style=flat-square" alt="RTL">
  <img src="https://img.shields.io/badge/Type-Case%20Study-111827?style=flat-square" alt="Case Study">
</p>

> ℹ️ Case study. The source code belongs to the client and is not included.

## Overview
**MidGram** connects the two sides of the pharmaceutical supply conversation. **Medical representatives** publish their products and reach pharmacies. **Pharmacies** discover reps, post what they need, chat directly and book visits. One app serves both audiences: the user picks a role on first launch and the app switches to a dedicated module for that role.

- **Market:** Iraq. Arabic-only, fully RTL, with an Iraqi city list and phone formats.
- **My role:** Flutter developer. I built the app end-to-end on Firebase: architecture, both role modules, realtime chat & appointments, auth, push notifications and remote configuration.
- **Distribution:** Delivered to the client. It is not on a public store listing at the time of writing.

## The challenge
- Pharmacies and reps coordinate over phone calls and personal messaging apps. Products, orders and visit times get lost in those chats.
- The two roles need **different home screens, navigation and data**, but share chat, appointments, profile and auth. Shipping two separate apps would double the maintenance.
- Messages and appointments must appear **in real time** on both phones, with unread counters and push notifications.
- The client needed control after release: force an update or switch on maintenance mode **without shipping a build**.

## Highlights
**💊 For pharmacies**
- Home feed of representatives and their product posts, with search.
- Representative directory and profile details. Pharmacies can also report a rep.
- **Create orders** (title, details of needed medicines, up to 5 images) and track them under *My Orders*.

**🧑‍💼 For medical representatives**
- Publish product posts (title, description, price, up to 5 images) and manage *My Posts*.
- Browse pharmacies and incoming orders, then open a pharmacy's details and start a chat.

**💬 Shared, realtime**
- **1-to-1 chat** with text and images (camera/gallery), unread counters and read receipts. Each participant can delete a conversation on their own side only.
- **Visit appointments** sent as a chat message with date & time. Both sides see them in a *Reminders* calendar, and either side can cancel.
- FCM push notifications for new messages and appointments.

**🔐 Platform**
- Role selection → Google sign-in or email/password → role-specific sign-up (pharmacy or representative, with city lists and unique-username checks).
- Remote **app configuration** in Firestore: minimum version (force-update screen with a store link), maintenance mode and store URLs.
- Deleted-account handling, privacy policy and terms of service screens, and a WhatsApp support entry for deactivated accounts.

## Screens
Real UI from the app, captured from a Flutter web build. Data screens are shown **empty on purpose**: everything behind login lives in the client's production database, and no demo data was invented.

<p align="center"><img src="assets/screens-onboarding.png" alt="Role selection, sign-up, privacy, force update" width="100%"></p>
<p align="center"><sub>Role selection (pharmacy / representative) · pharmacy sign-up · privacy policy · remote-config force-update screen</sub></p>

<p align="center"><img src="assets/screens-modules.png" alt="Pharmacy home, reps list, create order, chat, add post" width="100%"></p>
<p align="center"><sub>Pharmacy home · representatives directory · create order · chat composer with appointment & image actions · representative "publish product"</sub></p>

## Signature feature
<p align="center"><img src="assets/feature-two-sided.png" alt="Two-sided marketplace" width="100%"></p>

**Two roles, one codebase, one conversation.**

1. On first launch the user chooses **Pharmacy** or **Representative**. A `UserRoleService` stores that choice and every later route is resolved from it (`/pharmacy/...` or `/representative/...`), each with its own GetX **bindings**, controllers and bottom navbar.
2. Reps publish **products** and pharmacies publish **orders**, each with up to 5 images uploaded to Firebase Storage. Each side sees the other's content in its feed.
3. Either side opens a **chat**. `createOrGetChat` reuses an existing conversation between the two participants and reactivates it if one of them had deleted it earlier.
4. Inside the chat, a **visit appointment** is a typed message. Sending it also writes an appointment document, which shows up in both users' *Reminders*, and cancelling it updates both sides in real time.

## Architecture & stack
```mermaid
flowchart LR
  subgraph Modules["GetX modules (bindings · controllers · views)"]
    S[splash]
    AU[auth: role · login · sign-up · reset · update · maintenance · legal]
    PH[pharmacy: home · reps · orders · chat · reminders · profile]
    RE[representative: home · pharmacies · posts · chat · reminders · profile]
  end
  subgraph Services["Permanent services (Get.put)"]
    CU[CurrentUserService]
    RO[UserRoleService]
    DS[FirebaseDataService]
    CS[FirebaseChatService]
    CF[AppConfigService]
    AS[AuthService]
    FC[FCM service]
  end
  Modules --> Services
  AS --> FA[(Firebase Auth<br/>Google + email)]
  DS --> FS[(Cloud Firestore<br/>pharmacies · representatives · exhibits · orders · chats · messages · appointments · app_config)]
  CS --> FS
  CS --> ST[(Firebase Storage)]
  FC --> FM[(Firebase Cloud Messaging)]
```

| Layer | Tech |
|---|---|
| UI | Flutter, `flutter_screenutil`, `flutter_svg`, `google_fonts`, Cairo font, RTL `GetMaterialApp` |
| State, DI & routing | **GetX**: modular `bindings / controllers / views`, `GetPage` routes, `SmartManagement.keepFactory`, permanent services |
| Auth | `firebase_auth` + `google_sign_in`, email/password, role-aware auto-login |
| Data | `cloud_firestore` (realtime streams), `firebase_storage` for images, `cached_network_image` |
| Messaging | `firebase_messaging` push, in-chat appointments |
| Local | `shared_preferences` (role, cache), `package_info_plus` (version gate) |
| Device | `image_picker`, `permission_handler`, `url_launcher` |

## Engineering notes
- **Fast splash, parallel init.** Only Firebase, the cache and a few core services start before `runApp`. Auth, app config and app-init then run **in parallel** with a short minimum splash time, and any failure still navigates forward. Startup stays quick even on slow networks.
- **Role-aware routing.** Auto-login reads the Firebase user, loads the matching profile from `pharmacies` or `representatives` and routes to that module. A missing profile falls back to role selection, and a deleted account goes to a dedicated screen.
- **Remote kill-switches.** `AppConfigService` compares the installed version (`package_info_plus`) with `app_config` in Firestore. It can show a blocking **force-update** screen with the right store link, or a **maintenance** screen, all without a new release.
- **Chat that survives deletes.** Each conversation stores a per-participant status (`active` / `deleted`). Deleting hides the chat only for that user. A new message reactivates it for the recipient, and unread counts are kept per participant.
- **Appointments as first-class messages.** An appointment is sent through the chat service, persisted to its own collection and synced to both users' reminder screens through an integration helper that maps Firestore IDs to local reminders.
- **Arabic-first UI.** The whole app runs RTL with `Locale('ar')`, Cairo typography, Arabic validation messages and Iraqi city lists. Legal pages (privacy / terms) are built in-app.

## Results
- One Flutter app serving **2 user roles** (pharmacies and medical representatives) with shared realtime chat and appointments.
- Fully serverless on **Firebase** (Auth, Firestore, Storage, FCM), with remote force-update and maintenance controls.
- Delivered to the client. No public store listing is available to cite.

## بالعربي
**MidGram** منصة بتربط الصيدليات بالمندوبين الطبيين في العراق. المندوب ينشر منتجاته، والصيدلية تنشر طلباتها وتتواصل مع المندوب في شات لحظي.
من جوه الشات تقدر تحجز موعد زيارة، والموعد بيظهر عند الطرفين في صفحة التذكيرات.
تطبيق واحد بدورين، صيدلية ومندوب. كل دور ليه موديول خاص بيه، مبني بـ GetX (Bindings وControllers وViews).
الباك إند كله على Firebase: Auth وFirestore وStorage وFCM، وفيه تحكم عن بُعد لإجبار التحديث ووضع الصيانة. الواجهة عربي بالكامل RTL.

## Links
- Not publicly listed on the stores. Screens above are real UI captures.
- More of my work: [Portfolio](https://anashaker.github.io/shaker_portfolio/)

---
<h3 align="center">Built by Mahmoud Shaker — Senior Flutter Engineer</h3>
<p align="center">
  <a href="https://wa.me/201140279111"><img src="https://img.shields.io/badge/WhatsApp-Chat-25D366?style=for-the-badge&logo=whatsapp&logoColor=white" alt="WhatsApp"></a>
  <a href="https://www.linkedin.com/in/anashaker/"><img src="https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"></a>
  <a href="https://anashaker.github.io/shaker_portfolio/"><img src="https://img.shields.io/badge/Portfolio-Visit-111827?style=for-the-badge&logo=googlechrome&logoColor=white" alt="Portfolio"></a>
  <a href="mailto:mahmoudshaker382001@gmail.com"><img src="https://img.shields.io/badge/Email-Hire%20me-EA4335?style=for-the-badge&logo=gmail&logoColor=white" alt="Email"></a>
  <a href="https://khamsat.com/user/sheko201566"><img src="https://img.shields.io/badge/Khamsat-Services-F39C12?style=for-the-badge" alt="Khamsat"></a>
  <a href="https://mostaql.com/u/ShekoShaker"><img src="https://img.shields.io/badge/Mostaql-Freelance-2386C8?style=for-the-badge" alt="Mostaql"></a>
  <a href="https://github.com/ANaSHaker"><img src="https://img.shields.io/badge/GitHub-More%20work-181717?style=for-the-badge&logo=github&logoColor=white" alt="GitHub"></a>
</p>
