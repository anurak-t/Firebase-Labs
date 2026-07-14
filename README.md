# Lab Firebase NoSQL Minimal Social Network

เว็บ Minimal Social Network สำหรับรายวิชา **Database Modernization** หัวข้อ **Firebase NoSQL** ใช้ HTML, CSS และ JavaScript ธรรมดาเท่านั้น เพื่อให้นักศึกษาเห็นการทำงานของ Firebase Auth, Firestore, Realtime Database, Security Rules และ Hosting ในโปรเจกต์เดียว

## สิ่งที่ได้ทดลอง

- Google Authentication สำหรับเข้าสู่ระบบ
- Firestore สำหรับเก็บข้อมูล profile, feed, post, comment และ like
- Realtime Database สำหรับจัดการ presence, typing indicator และ direct message chat
- Security Rules สำหรับแยกสิทธิ์อ่าน/เขียนตาม `auth.uid`
- Firebase Hosting สำหรับ deploy static web app

## โครงสร้างไฟล์

```text
.
├── public/
│   ├── index.html
│   ├── styles.css
│   ├── app.js
│   └── firebase-config.example.js
├── firebase.json
├── firestore.rules
├── database.rules.json
├── .firebaserc.example
└── README.md
```

## 1. เตรียม Firebase Project

1. เข้า Firebase Console แล้วสร้าง project ใหม่
2. เพิ่ม Web App ใน project
3. คัดลอก Firebase config ที่มีค่า `apiKey`, `authDomain`, `databaseURL`, `projectId`, `storageBucket`, `messagingSenderId`, และ `appId`
4. เปิดเมนู Authentication แล้ว enable provider แบบ Google
5. เปิด Cloud Firestore โดยเริ่มจาก production mode ได้ เพราะเราจะ deploy rules เอง
6. เปิด Realtime Database และเลือก region ที่เหมาะกับห้องเรียน เช่น `asia-southeast1`

## 2. ตั้งค่าไฟล์ Config

คัดลอกไฟล์ตัวอย่าง:

```powershell
Copy-Item public/firebase-config.example.js public/firebase-config.js
```

แก้ `public/firebase-config.js` ให้เป็น config ของ project ตัวเอง:

```js
export const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  databaseURL: "https://YOUR_PROJECT_ID-default-rtdb.asia-southeast1.firebasedatabase.app",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_APP_ID"
};

export const useEmulators = false;
```


## 3. ติดตั้งและ Login Firebase CLI

ติดตั้ง CLI ถ้ายังไม่มี:

```powershell
npm install -g firebase-tools
```

เข้าสู่ระบบ:

```powershell
firebase login
```

ตั้งค่า project:

```powershell
Copy-Item .firebaserc.example .firebaserc
```

จากนั้นแก้ `.firebaserc` ให้ `your-firebase-project-id` เป็น project ID จริง หรือใช้คำสั่ง:

```powershell
firebase use --add
```

## 4. แนวคิดของ `firebase init`

โปรเจกต์นี้เตรียมไฟล์ที่ปกติ `firebase init` จะสร้างไว้แล้ว:

- `firebase.json` กำหนด Hosting public folder เป็น `public` และชี้ rules files
- `firestore.rules` คือ rules ของ Cloud Firestore
- `database.rules.json` คือ rules ของ Realtime Database
- `.firebaserc` ผูกโฟลเดอร์นี้กับ Firebase project

ถ้าต้อง init เองใน lab ใหม่ ให้เลือก Hosting, Firestore และ Realtime Database แล้วตั้ง public directory เป็น `public`

## 5. ทดสอบบนเครื่อง

รัน static hosting ในเครื่อง:

```powershell
firebase serve --only hosting
```

หรือใช้ emulator suite:

```powershell
firebase emulators:start --only hosting,firestore,database,auth
```

ถ้าจะใช้ emulator ให้เปลี่ยน `useEmulators` ใน `public/firebase-config.js` เป็น `true` แต่ Google popup sign-in เหมาะกับการทดสอบกับ Firebase project จริงมากกว่า

## 6. Deploy Rules และ Hosting

Deploy rules และเว็บพร้อมกัน:

```powershell
firebase deploy --only firestore:rules,database,hosting
```

หลัง deploy เสร็จ CLI จะแสดง Hosting URL เช่น:

```text
https://YOUR_PROJECT_ID.web.app
```

ให้นักศึกษาเปิด URL นี้ ทดสอบ sign in, post, like, comment และ chat

## การออกแบบข้อมูล

Firestore เหมาะกับข้อมูล social feed เพราะ query เอกสารแบบเรียงเวลาได้ดี หรือ order by ใน relational database นั้นเอง และทำ subcollection สำหรับ comments/likes ได้ชัด โดยไม่ต้องทำ table เพิ่มและ join table เหมือน relational database:

```text
users/{uid}
posts/{postId}
posts/{postId}/comments/{commentId}
posts/{postId}/likes/{uid}
```

Realtime Database เหมาะกับข้อมูลที่ต้อง realtime และมี state เปลี่ยนบ่อย เช่น online status, typing และ chat messages:

```text
status/{uid}
chats/{chatId}
chatMessages/{chatId}/{messageId}
typing/{chatId}/{uid}
```

DM chat ใช้ deterministic chat ID จาก UID สองคนที่ sort แล้ว เช่น:

```text
uidA_uidB
```

วิธีนี้ทำให้ผู้ใช้คู่เดิมเปิดห้องเดิมเสมอ ไม่ต้อง query หา room ซ้ำ

## เทคนิคที่ควรสังเกต

- ใช้ `auth.uid` เป็น key สำคัญของ rules เพื่อให้ตรวจสิทธิ์ง่าย
- เก็บ `authorName` และ `authorPhotoURL` ซ้ำใน post/comment เป็น denormalized author snapshot เพื่อ render feed ได้เร็วโดยไม่ต้อง join
- ใช้ Firestore `onSnapshot` กับ feed เพื่อเห็น document updates แบบ realtime
- ใช้ RTDB `onValue`, `push`, `set`, `update` และ `onDisconnect` กับ chat/presence
- ใช้ Hosting เสิร์ฟไฟล์ static ทั้งหมด จึงไม่ต้องมี backend server
- Rules ตั้งใจให้เขียนผิดสิทธิ์แล้ว fail เพื่อใช้เป็นโจทย์ทดลอง

## Firebase SDK Syntax ที่ใช้บ่อย

ก่อนทำแบบฝึกหัด ให้นักศึกษาทำความเข้าใจ function หลักของ Firebase SDK ก่อน เพราะ code ส่วนใหญ่ใน lab นี้เป็นการประกอบ `reference` แล้วค่อยสั่งอ่าน/เขียนข้อมูล

### แนวคิดสำคัญ: Reference ก่อน Action

Firebase SDK มักทำงาน 2 ขั้น:

1. สร้าง reference ไปยังตำแหน่งข้อมูล เช่น document, collection หรือ RTDB path
2. ส่ง reference นั้นให้ function ที่อ่าน/เขียน/subscribe

ตัวอย่าง:

```js
const userRef = doc(state.db, "users", user.uid);
const snapshot = await getDoc(userRef);
```

บรรทัดแรกยังไม่ได้อ่านข้อมูล เป็นแค่การชี้ตำแหน่ง `users/{uid}` ส่วนบรรทัดที่สองจึงเป็นการอ่านจริง

## Firestore Syntax

Firestore เก็บข้อมูลเป็น document ใน collection เหมาะกับข้อมูลที่ต้อง query เช่น users, posts, comments และ likes

### `collection(db, path)`

ใช้สร้าง reference ไปยัง collection

```js
const postsRef = collection(state.db, "posts");
```

ใช้เมื่อจะ:

- เพิ่ม document ใหม่ด้วย `addDoc`
- สร้าง query ด้วย `query`
- subscribe collection ด้วย `onSnapshot`

### `doc(db, path, id)` หรือ `doc(collectionRef, id)`

ใช้สร้าง reference ไปยัง document ที่ระบุ ID ชัดเจน

```js
const userRef = doc(state.db, "users", user.uid);
const likeRef = doc(state.db, "posts", postId, "likes", state.user.uid);
```

ใช้เมื่อรู้ path ชัดเจน เช่น:

- user profile: `users/{uid}`
- like: `posts/{postId}/likes/{uid}`
- post ที่ต้อง delete: `posts/{postId}`

### `addDoc(collectionRef, data)`

ใช้เพิ่ม document ใหม่โดยให้ Firestore สร้าง random ID ให้อัตโนมัติ

```js
await addDoc(collection(state.db, "posts"), {
  authorId: state.user.uid,
  text,
  createdAt: serverTimestamp()
});
```

เหมาะกับข้อมูลที่มีหลายรายการและไม่ต้องกำหนด ID เอง เช่น posts หรือ comments

### `setDoc(docRef, data)`

ใช้เขียนข้อมูลลง document ID ที่กำหนดเอง ถ้า document ยังไม่มีจะสร้างใหม่ ถ้ามีอยู่แล้วจะเขียนทับทั้ง document

```js
await setDoc(doc(state.db, "posts", postId, "likes", state.user.uid), {
  uid: state.user.uid,
  createdAt: serverTimestamp()
});
```

เหมาะกับข้อมูลที่ต้องกัน duplicate โดยใช้ ID เดิม เช่น like 1 user ต่อ 1 post

ถ้าต้องการ merge ไม่ให้ทับ field เดิมทั้งหมด ใช้ option:

```js
await setDoc(userRef, profile, { merge: true });
```

### `updateDoc(docRef, partialData)`

ใช้ update บาง field ของ document ที่มีอยู่แล้ว ถ้า document ไม่มีจะ error

```js
await updateDoc(userRef, {
  displayName: user.displayName,
  photoURL: user.photoURL,
  updatedAt: serverTimestamp()
});
```

เหมาะกับการ update profile, แก้ข้อความ, หรือแก้ field เฉพาะจุด

จำง่าย:

| Function | สร้างใหม่ได้ไหม | ทับทั้ง document ไหม | ใช้เมื่อ |
| --- | --- | --- | --- |
| `addDoc` | ได้ | ไม่เกี่ยว เพราะสร้าง ID ใหม่ | เพิ่มรายการใหม่ |
| `setDoc` | ได้ | ใช่ ถ้าไม่ใส่ `{ merge: true }` | เขียน document ID ที่รู้แน่นอน |
| `updateDoc` | ไม่ได้ | ไม่ ทับเฉพาะ field ที่ส่งไป | แก้บาง field ของ document เดิม |

### `getDoc(docRef)`

ใช้อ่าน document ครั้งเดียว

```js
const snapshot = await getDoc(userRef);

if (snapshot.exists()) {
  const data = snapshot.data();
}
```

ใช้เมื่อต้องเช็กก่อนว่า document มีอยู่ไหม เช่น profile หรือ like

### `deleteDoc(docRef)`

ใช้ลบ document

```js
await deleteDoc(doc(state.db, "posts", postId));
```

ใน lab นี้ใช้กับการลบ post, comment และ unlike

### `query(collectionRef, ...constraints)`

ใช้สร้าง query จาก collection

```js
const postsQuery = query(
  collection(state.db, "posts"),
  orderBy("createdAt", "desc"),
  limit(30)
);
```

function ที่ใช้ร่วมกันบ่อย:

- `orderBy("createdAt", "desc")` เรียงจากใหม่ไปเก่า
- `orderBy("createdAt", "asc")` เรียงจากเก่าไปใหม่
- `limit(30)` จำกัดจำนวน document

### `onSnapshot(refOrQuery, callback)`

ใช้ subscribe Firestore แบบ realtime เมื่อข้อมูลเปลี่ยน callback จะทำงานใหม่

```js
const unsubscribe = onSnapshot(postsQuery, (snapshot) => {
  snapshot.forEach((postDoc) => {
    console.log(postDoc.id, postDoc.data());
  });
});
```

ควรเก็บ `unsubscribe` ไว้เรียกตอนออกจากหน้า หรือก่อน subscribe ใหม่

```js
if (state.unsubscribePosts) state.unsubscribePosts();
state.unsubscribePosts = onSnapshot(postsQuery, renderFeed);
```

### `serverTimestamp()`

ใช้ให้ server เป็นคนใส่เวลาจริง ลดปัญหาเวลาเครื่อง client ไม่ตรงกัน

```js
await addDoc(collection(state.db, "posts"), {
  text,
  createdAt: serverTimestamp()
});
```

## Realtime Database Syntax

Realtime Database เก็บข้อมูลเป็น JSON tree เหมาะกับข้อมูลที่เปลี่ยนเร็ว เช่น online status, typing และ chat

### `ref(db, path)`

ใช้สร้าง reference ไปยัง path ใน RTDB

```js
const statusRef = ref(state.rtdb, `status/${state.user.uid}`);
const messagesRef = ref(state.rtdb, `chatMessages/${chatId}`);
```

ต่างจาก Firestore คือ RTDB ใช้ path แบบ JSON tree ไม่ใช่ collection/document

### `set(ref, value)`

เขียนค่าทับ path นั้นทั้งก้อน

```js
await set(ref(state.rtdb, `typing/${chatId}/${state.user.uid}`), true);
```

เหมาะกับค่าเดี่ยวหรือ object ที่ path ชัดเจน เช่น typing หรือ status

### `update(ref, object)`

update บาง field หรือหลาย path พร้อมกัน

```js
await update(ref(state.rtdb, `chats/${chatId}`), {
  lastMessage: text.slice(0, 160),
  lastSenderId: state.user.uid,
  updatedAt: Date.now()
});
```

สามารถ update หลาย path จาก root ได้:

```js
await update(ref(state.rtdb), {
  [`userChats/${myUid}/${chatId}/unread`]: false,
  [`chats/${chatId}/updatedAt`]: Date.now()
});
```

### `push(ref, value)`

เพิ่ม child ใหม่พร้อม key อัตโนมัติ เหมาะกับ chat messages

```js
await push(ref(state.rtdb, `chatMessages/${chatId}`), {
  senderId: state.user.uid,
  text,
  createdAt: Date.now()
});
```

ใช้ `push` เมื่อข้อมูลเป็นรายการต่อเนื่อง และไม่อยากกำหนด key เอง

### `get(ref)`

อ่านข้อมูลจาก RTDB ครั้งเดียว

```js
const snapshot = await get(ref(state.rtdb, `chats/${chatId}`));

if (snapshot.exists()) {
  const chat = snapshot.val();
}
```

ใช้เมื่ออยากเช็กว่าห้องแชตมีอยู่แล้วหรือยัง

### `onValue(ref, callback)`

subscribe ข้อมูล RTDB แบบ realtime

```js
const unsubscribe = onValue(ref(state.rtdb, "status"), (snapshot) => {
  const statuses = snapshot.val() || {};
  console.log(statuses);
});
```

ใช้กับ presence, chat messages, typing และ inbox notification

### `off(ref)`

ยกเลิก listener ของ RTDB reference

```js
if (state.chatMessagesRef) off(state.chatMessagesRef);
```

ใน lab นี้ใช้ตอนปิดห้องแชตหรือเปลี่ยนไปคุยกับคนอื่น

### `onDisconnect(ref).set(value)`

สั่งให้ Firebase server เขียนค่าให้เมื่อ client disconnect

```js
onDisconnect(statusRef)
  .set({
    state: "offline",
    lastChanged: rtdbServerTimestamp()
  })
  .then(() => set(statusRef, {
    state: "online",
    lastChanged: rtdbServerTimestamp()
  }));
```

ใช้ทำ online/offline presence เพราะถ้าปิด browser ทันที client อาจไม่มีโอกาสส่งคำสั่ง offline เอง

### `rtdbServerTimestamp()`

ใช้ timestamp ของ RTDB server

```js
import { serverTimestamp as rtdbServerTimestamp } from "firebase-database";
```

ในโปรเจกต์นี้ import จาก CDN:

```js
import {
  serverTimestamp as rtdbServerTimestamp
} from "https://www.gstatic.com/firebasejs/12.16.0/firebase-database.js";
```

## Auth Syntax

### `onAuthStateChanged(auth, callback)`

ทำงานเมื่อสถานะ login เปลี่ยน เช่น login สำเร็จ, logout, refresh หน้าแล้วยังมี session

```js
onAuthStateChanged(state.auth, (user) => {
  if (!user) {
    // show login
    return;
  }

  // show app
});
```

### `signInWithPopup(auth, provider)`

เปิด popup login ด้วย provider เช่น Google

```js
const provider = new GoogleAuthProvider();
await signInWithPopup(state.auth, provider);
```

### `signOut(auth)`

ออกจากระบบ

```js
await signOut(state.auth);
```

## รูปแบบ Code ที่เจอบ่อยใน Lab

### 1. สร้าง reference แล้วอ่านครั้งเดียว

```js
const userRef = doc(state.db, "users", state.user.uid);
const snapshot = await getDoc(userRef);
```

### 2. สร้าง reference แล้ว subscribe realtime

```js
const statusesRef = ref(state.rtdb, "status");
state.unsubscribeStatuses = onValue(statusesRef, (snapshot) => {
  const statuses = snapshot.val() || {};
});
```

### 3. เขียนข้อมูลพร้อม timestamp

```js
await addDoc(collection(state.db, "posts"), {
  text,
  authorId: state.user.uid,
  createdAt: serverTimestamp()
});
```

### 4. เขียนข้อมูลเข้าหลาย RTDB path พร้อมกัน

```js
await update(ref(state.rtdb), {
  [`userChats/${senderUid}/${chatId}`]: senderInbox,
  [`userChats/${receiverUid}/${chatId}`]: receiverInbox
});
```

### 5. เก็บ unsubscribe เพื่อป้องกัน listener ซ้ำ

```js
if (state.unsubscribePosts) state.unsubscribePosts();
state.unsubscribePosts = onSnapshot(postsQuery, renderPosts);
```

## เปรียบเทียบ Firestore กับ Realtime Database แบบเร็ว

| เรื่อง | Firestore | Realtime Database |
| --- | --- | --- |
| โครงสร้างข้อมูล | Collection / Document | JSON Tree |
| Reference | `doc()`, `collection()` | `ref()` |
| อ่านครั้งเดียว | `getDoc()` | `get()` |
| realtime listener | `onSnapshot()` | `onValue()` |
| เพิ่มรายการใหม่ | `addDoc()` | `push()` |
| เขียนทับ path/document | `setDoc()` | `set()` |
| update บาง field | `updateDoc()` | `update()` |
| timestamp server | `serverTimestamp()` | `rtdbServerTimestamp()` |
| เหมาะกับ | feed, profile, query | presence, typing, chat, notification |

## แบบฝึกหัดในห้องเรียน

### สิ่งที่ผู้ทำ lab ควรได้รับ

หลังจบ lab นักศึกษาควรอธิบายและทำได้ 3 เรื่องหลัก:

1. ทำ CRUD และ query ด้วย Firestore และ Realtime Database
2. เขียน Security Rules แบบง่าย โดยใช้ `auth.uid` แยกสิทธิ์ของผู้ใช้
3. เข้าใจ realtime events เช่น `onSnapshot`, `onValue`, `onDisconnect`, typing event และ notification event

### วิธีส่งงาน

ให้นักศึกษาส่ง:

- URL จาก Firebase Hosting
- screenshot ข้อมูลใน Firestore และ Realtime Database
- คำตอบสั้น ๆ ว่า Firestore กับ Realtime Database เหมาะกับข้อมูลคนละแบบอย่างไร
- คำอธิบาย bug ที่เจอและวิธีแก้

## Lab 1: Authentication และ User Profile

เป้าหมาย: เมื่อนักศึกษา login ด้วย Google แล้วต้องสร้างหรือ update เอกสาร `users/{uid}` ใน Firestore

```js
async function ensureUserProfile(user) {
  const userRef = doc(state.db, "users", user.uid);

  // TODO: 1) อ่าน document users/{uid}
  // Guide: ใช้ getDoc(userRef)
  const snapshot = /* fill code */;

  const profile = {
    uid: user.uid,
    displayName: user.displayName || "Member",
    email: user.email || "",
    photoURL: user.photoURL || avatarFallback(user.displayName),
    updatedAt: serverTimestamp()
  };

  // TODO: 2) ถ้ามี profile อยู่แล้ว ให้ update ข้อมูลล่าสุด
  // Guide: ใช้ snapshot.exists() และ updateDoc(userRef, profile)
  if (/* fill condition */) {
    /* fill code */
    return;
  }

  // TODO: 3) ถ้ายังไม่มี profile ให้สร้าง document ใหม่พร้อม createdAt
  // Guide: ใช้ setDoc(userRef, { ...profile, createdAt: serverTimestamp() })
  /* fill code */
}
```

เฉลย:

```js
async function ensureUserProfile(user) {
  const userRef = doc(state.db, "users", user.uid);
  const snapshot = await getDoc(userRef);
  const profile = {
    uid: user.uid,
    displayName: user.displayName || "Member",
    email: user.email || "",
    photoURL: user.photoURL || avatarFallback(user.displayName),
    updatedAt: serverTimestamp()
  };

  if (snapshot.exists()) {
    await updateDoc(userRef, profile);
    return;
  }

  await setDoc(userRef, {
    ...profile,
    createdAt: serverTimestamp()
  });
}
```

คำถามท้าย lab:

- ทำไม document ID ของ user ควรเป็น `uid` จาก Firebase Auth
- ถ้าใช้ random document ID แทน จะเขียน rules ยากขึ้นอย่างไร

## Lab 2: Firestore CRUD สำหรับ Post

เป้าหมาย: สร้าง, อ่าน, update/like และ delete post/comment ด้วย Firestore

### 2.1 Create Post

ตำแหน่งที่ให้เติม: `labs/starter/firebase-tasks.js` (เมื่อทำเสร็จจึงคัดลอกไป `public/app.js`):

```js
async function createPost(event) {
  event.preventDefault();
  const text = els.postText.value.trim();
  if (!text || !state.user) return;

  // TODO: เพิ่ม document ใหม่ใน collection posts
  // Guide:
  // - ใช้ addDoc(collection(state.db, "posts"), data)
  // - เก็บ authorId เป็น state.user.uid
  // - createdAt และ updatedAt ใช้ serverTimestamp()
  await /* fill code */;

  els.postText.value = "";
  els.postCount.textContent = "0/1000";
}
```

เฉลย:

```js
await addDoc(collection(state.db, "posts"), {
  authorId: state.user.uid,
  authorName: state.user.displayName || "Member",
  authorPhotoURL: state.user.photoURL || avatarFallback(state.user.displayName),
  text,
  visibility: "public",
  createdAt: serverTimestamp(),
  updatedAt: serverTimestamp()
});
```

### 2.2 Query Feed แบบ realtime

ตำแหน่งที่ให้เติม: `labs/starter/firebase-tasks.js`:

```js
function subscribePosts() {
  if (state.unsubscribePosts) state.unsubscribePosts();
  clearPostListeners();

  // TODO: query posts ล่าสุด 30 รายการ เรียง createdAt จากใหม่ไปเก่า
  // Guide: ใช้ query(), collection(), orderBy("createdAt", "desc"), limit(30)
  const postsQuery = /* fill code */;

  // TODO: subscribe ด้วย onSnapshot แล้ว render post ทุก document
  state.unsubscribePosts = onSnapshot(postsQuery, (snapshot) => {
    els.feed.innerHTML = "";

    // Guide: ถ้า snapshot.empty ให้แสดง empty state
    if (/* fill condition */) {
      els.feed.innerHTML = `<article class="post empty-state">ยังไม่มีโพสต์</article>`;
      return;
    }

    // Guide: ใช้ snapshot.forEach((postDoc) => renderPost(postDoc.id, postDoc.data()))
    /* fill code */
  });
}
```

เฉลย:

```js
const postsQuery = query(
  collection(state.db, "posts"),
  orderBy("createdAt", "desc"),
  limit(30)
);

state.unsubscribePosts = onSnapshot(postsQuery, (snapshot) => {
  els.feed.innerHTML = "";

  if (snapshot.empty) {
    els.feed.innerHTML = `<article class="post empty-state">ยังไม่มีโพสต์</article>`;
    return;
  }

  snapshot.forEach((postDoc) => renderPost(postDoc.id, postDoc.data()));
});
```

### 2.3 Like ด้วย document ID เป็น UID

โจทย์: เติม code ให้กด like แล้วสร้าง document ที่ `posts/{postId}/likes/{uid}` ถ้ามีอยู่แล้วให้ลบออก

```js
async function toggleLike(postId) {
  // TODO: สร้าง reference ไปที่ posts/{postId}/likes/{uid}
  const likeRef = /* fill code */;

  // TODO: อ่าน like document
  const snapshot = /* fill code */;

  // TODO: ถ้ามี document แล้วให้ลบออก
  if (/* fill condition */) {
    /* fill code */
    return;
  }

  // TODO: ถ้ายังไม่มี ให้ setDoc พร้อม uid และ createdAt
  /* fill code */
}
```

เฉลย:

```js
async function toggleLike(postId) {
  const likeRef = doc(state.db, "posts", postId, "likes", state.user.uid);
  const snapshot = await getDoc(likeRef);

  if (snapshot.exists()) {
    await deleteDoc(likeRef);
    return;
  }

  await setDoc(likeRef, {
    uid: state.user.uid,
    createdAt: serverTimestamp()
  });
}
```

คำถามท้าย lab:

- ทำไม like ใช้ `setDoc` กับ document ID เป็น `uid` แทน `addDoc`
- ถ้าใช้ `addDoc` จะเกิด duplicate like ได้อย่างไร

## Lab 3: Realtime Database สำหรับ Presence และ Chat

เป้าหมาย: ใช้ RTDB สำหรับข้อมูลที่เปลี่ยนเร็ว เช่น online/offline, typing และ chat messages

### 3.1 Online Status ด้วย `onDisconnect`

ตำแหน่งที่ให้เติม: `labs/starter/firebase-tasks.js`:

```js
function setupPresence(user) {
  const statusRef = ref(state.rtdb, `status/${user.uid}`);
  const connectedRef = ref(state.rtdb, ".info/connected");

  const online = {
    state: "online",
    lastChanged: rtdbServerTimestamp(),
    displayName: user.displayName || "Member",
    photoURL: user.photoURL || avatarFallback(user.displayName)
  };

  const offline = {
    ...online,
    state: "offline",
    lastChanged: rtdbServerTimestamp()
  };

  state.unsubscribeConnection = onValue(connectedRef, (snapshot) => {
    if (snapshot.val() === false) return;

    // TODO: ตั้งค่าให้ Firebase เขียน offline อัตโนมัติเมื่อ disconnect
    // Guide: ใช้ onDisconnect(statusRef).set(offline)
    // TODO: หลัง register onDisconnect สำเร็จ ให้ set online
    /* fill code */
  });
}
```

เฉลย:

```js
state.unsubscribeConnection = onValue(connectedRef, (snapshot) => {
  if (snapshot.val() === false) return;

  onDisconnect(statusRef)
    .set(offline)
    .then(() => set(statusRef, online));
});
```

### 3.2 อ่านรายชื่อ status ทั้งหมด

โจทย์: เติม listener ให้ app เห็นว่าเพื่อนคนไหน online/offline

```js
function subscribeStatuses() {
  const statusesRef = ref(state.rtdb, "status");

  // TODO: ใช้ onValue เพื่ออ่าน status ทั้งหมด
  // Guide: snapshot.val() จะได้ object เช่น { uid1: { state: "online" } }
  state.unsubscribeStatuses = onValue(statusesRef, (snapshot) => {
    state.statuses.clear();
    const statuses = /* fill code */;

    // TODO: แปลง object เป็น Map
    Object.entries(statuses).forEach(([uid, value]) => {
      /* fill code */
    });

    renderPeople();
  });
}
```

เฉลย:

```js
state.unsubscribeStatuses = onValue(statusesRef, (snapshot) => {
  state.statuses.clear();
  const statuses = snapshot.val() || {};

  Object.entries(statuses).forEach(([uid, value]) => {
    state.statuses.set(uid, value);
  });

  renderPeople();
});
```

### 3.3 ส่ง Chat Message

โจทย์: เติม code ให้ส่งข้อความเข้า `chatMessages/{chatId}` และ update summary ที่ `chats/{chatId}`

```js
async function sendChatMessage(event) {
  event.preventDefault();
  if (!state.activeChat) return;

  const text = els.chatInput.value.trim();
  if (!text) return;

  const now = Date.now();
  const message = {
    senderId: state.user.uid,
    senderName: state.user.displayName || "Member",
    senderPhotoURL: state.user.photoURL || avatarFallback(state.user.displayName),
    text,
    createdAt: now
  };

  els.chatInput.value = "";

  // TODO: push message ไปที่ chatMessages/{chatId}
  /* fill code */

  // TODO: update lastMessage, lastSenderId, updatedAt ที่ chats/{chatId}
  /* fill code */
}
```

เฉลย:

```js
await push(ref(state.rtdb, `chatMessages/${state.activeChat.chatId}`), message);
await update(ref(state.rtdb, `chats/${state.activeChat.chatId}`), {
  lastMessage: text.slice(0, 160),
  lastSenderId: state.user.uid,
  updatedAt: now
});
```

## Lab 4: Security Rules แบบง่าย

เป้าหมาย: เข้าใจว่า client code ไม่พอ ต้องมี rules คุมสิทธิ์ที่ฝั่ง Firebase ด้วย

### 4.1 Firestore Rules

โจทย์: เติม rules ให้ผู้ใช้ login แล้วอ่าน posts ได้ แต่ลบ post ได้เฉพาะเจ้าของ

```js
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {
    match /posts/{postId} {
      // TODO: ให้ user ที่ login แล้วอ่าน posts ได้
      allow read: if /* fill condition */;

      // TODO: ให้ user ที่ login แล้วสร้าง post ได้ และ authorId ต้องเป็น uid ตัวเอง
      allow create: if /* fill condition */;

      // TODO: ให้ลบได้เฉพาะเจ้าของ post
      allow delete: if /* fill condition */;
    }
  }
}
```

เฉลย:

```js
allow read: if request.auth != null;
allow create: if request.auth != null
  && request.resource.data.authorId == request.auth.uid;
allow delete: if request.auth != null
  && resource.data.authorId == request.auth.uid;
```

### 4.2 Realtime Database Rules

โจทย์: เติม rules ให้ user เขียน online status ได้เฉพาะ path ของตัวเอง

```json
{
  "rules": {
    "status": {
      ".read": "auth != null",
      "$uid": {
        ".write": "/* fill condition */",
        ".validate": "newData.hasChildren(['state', 'lastChanged'])"
      }
    }
  }
}
```

เฉลย:

```json
".write": "auth != null && auth.uid === $uid"
```

คำถามท้าย lab:

- `request.auth.uid` ใน Firestore Rules ต่างจาก `auth.uid` ใน RTDB Rules อย่างไร
- ถ้าเปิด `.read: true` และ `.write: true` ทั้ง project มีความเสี่ยงอะไร
- ทำไม app นี้ให้ทุกคนที่ login อ่าน `/status` ได้ แต่เขียนได้เฉพาะ `status/{uid}` ของตัวเอง

## Lab 5: Event และ Feature เสริม

เป้าหมาย: เห็นความต่างของ event ใน Firestore และ RTDB

ให้นักศึกษาจับคู่ event กับ use case:

| Event/API | ใช้กับ | เหตุผล |
| --- | --- | --- |
| `onSnapshot` | Feed, comments, likes | Firestore query และ subcollection เปลี่ยนแล้ว render ใหม่ |
| `onValue` | Online status, chat messages, typing | RTDB path เปลี่ยนแล้วได้ค่าใหม่ทันที |
| `onDisconnect` | Presence | ให้ server เขียน offline ตอน client หลุด |
| `push` | Chat messages | สร้าง key ใหม่ตามเวลาโดยอัตโนมัติ |
| `set` | Like, typing, status | เขียนค่าที่ path ชัดเจน |
| `update` | Chat summary, profile update | update บาง field โดยไม่ทับทั้ง object |

### Challenge: Chat Notification

โจทย์: เมื่อมีข้อความจากเพื่อนเข้ามา ให้แสดง notification เล็ก ๆ และทำ unread badge ในรายชื่อเพื่อน

แนวทาง:

```js
function subscribeInbox() {
  // TODO: ฟัง userChats/{currentUid}
  // Guide: path นี้เป็น inbox ส่วนตัวของ user ที่ login อยู่

  // TODO: ถ้า lastSenderId ไม่ใช่ uid ของเรา และ unread เป็น true ให้ show notification

  // TODO: เมื่อเปิด chat แล้วให้ set unread เป็น false
}
```

เฉลยแนวคิด:

```js
state.unsubscribeInbox = onValue(ref(state.rtdb, `userChats/${state.user.uid}`), (snapshot) => {
  const inbox = snapshot.val() || {};

  Object.entries(inbox).forEach(([chatId, chat]) => {
    const isIncoming = chat.lastSenderId && chat.lastSenderId !== state.user.uid;
    const isActiveChat = state.activeChat?.chatId === chatId;

    if (isIncoming && chat.unread && !isActiveChat) {
      showChatNotification(chat, chatId);
    }
  });
});
```

## Lab 6: Bug Hunt

ผู้สอนสามารถตั้งใจสร้าง bug เล็ก ๆ แล้วให้นักศึกษา debug จากอาการ, console error, data path และ rules

### Bug 1: Status ไม่ update ทุกคนขึ้น offline

อาการ:

- login สำเร็จ
- ตัวเองหรือเพื่อนยังขึ้น `offline`
- ใน console อาจมี permission denied จาก Realtime Database

starter bug ใน `database.rules.json`:

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "status": {
      "$uid": {
        ".read": "auth != null",
        ".write": "auth != null && auth.uid === $uid"
      }
    }
  }
}
```

คำถาม:

- โค้ดอ่าน path ไหน
- rules อนุญาตอ่าน path ไหน
- ทำไมอ่าน `status/{uid}` ได้ แต่การอ่าน `/status` ทั้งก้อนล้มเหลว

เฉลย:

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "status": {
      ".read": "auth != null",
      "$uid": {
        ".write": "auth != null && auth.uid === $uid"
      }
    }
  }
}
```

สรุป: ถ้า client ใช้ `ref(state.rtdb, "status")` rules ต้องอนุญาตอ่านที่ node `status` ด้วย ไม่ใช่อนุญาตเฉพาะ child

### Bug 2: Like ซ้ำได้หลายครั้ง

starter bug:

```js
await addDoc(collection(state.db, "posts", postId, "likes"), {
  uid: state.user.uid,
  createdAt: serverTimestamp()
});
```

คำถาม:

- กด like 3 ครั้งเกิด document กี่อัน
- ทำไม document ID แบบ random ทำให้กัน duplicate ยาก

เฉลย:

```js
const likeRef = doc(state.db, "posts", postId, "likes", state.user.uid);
await setDoc(likeRef, {
  uid: state.user.uid,
  createdAt: serverTimestamp()
});
```

### Bug 3: เปิด chat แล้ว last message หาย

starter bug:

```js
await update(ref(state.rtdb, `chats/${chatId}`), {
  members: {
    [state.user.uid]: true,
    [peerUid]: true
  },
  updatedAt: Date.now(),
  lastMessage: "",
  lastSenderId: ""
});
```

คำถาม:

- ทำไมเปิดห้องแชตเดิมแล้วข้อความล่าสุดกลายเป็นค่าว่าง
- field ไหนควร set เฉพาะตอนสร้างห้องใหม่เท่านั้น

เฉลยแนวคิด:

```js
const chatRef = ref(state.rtdb, `chats/${chatId}`);
const chatSnapshot = await get(chatRef);
const chatUpdate = {
  [`members/${state.user.uid}`]: true,
  [`members/${peerUid}`]: true,
  updatedAt: Date.now()
};

if (!chatSnapshot.exists()) {
  chatUpdate.createdAt = Date.now();
  chatUpdate.lastMessage = "";
  chatUpdate.lastSenderId = "";
}

await update(chatRef, chatUpdate);
```
