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

- TODO: 1. อ่าน document users/{uid}
- TODO: 2. ถ้ามี profile อยู่แล้ว ให้ update ข้อมูลล่าสุด
- TODO: 3. ถ้ายังไม่มี profile ให้สร้าง document ใหม่พร้อม createdAt

## Lab 2: Firestore CRUD สำหรับ Post

เป้าหมาย: สร้าง, อ่าน, update/like และ delete post/comment ด้วย Firestore

### 2.1 Create Post

- TODO: 4. เพิ่ม document ใหม่ใน collection posts ด้วย postData

### 2.2 Query Feed แบบ realtime

- TODO: 5. query posts ล่าสุด 30 รายการ เรียง createdAt จากใหม่ไปเก่า

### 2.3 Like ด้วย document ID เป็น UID

- TODO: 6. สร้าง reference ไปที่ posts/{postId}/likes/{uid}
- TODO: 7. อ่าน like document
- TODO: 8. ถ้ามี document แล้วให้ลบออก

## Lab 3: Realtime Database สำหรับจัดการ Presence และ Chat

เป้าหมาย: ใช้ RTDB สำหรับข้อมูลที่เปลี่ยนเร็ว เช่น online/offline, typing และ chat messages

### 3.1 Online Status ด้วย `onDisconnect`

- TODO: 9. ตั้งค่าให้ Firebase เขียน offline อัตโนมัติเมื่อ disconnect และ set ค่าเริ่มต้นเป็น online

### 3.3 ส่ง Chat Message

- TODO: 10. เติม code ให้ส่งข้อความเข้า `chatMessages/{chatId}`

## Lab 4: Security Rules แบบง่าย

เป้าหมาย: เข้าใจว่า client code ไม่พอ ต้องมี rules คุมสิทธิ์ที่ฝั่ง Firebase ด้วย

### 4.1 Firestore Rules

โจทย์: เติม rules ผู้ใช้ที่ login แล้วสามารถอ่าน posts ได้ทะ้งหมด แต่ลบ post ไม่ได้ ได้เฉพาะเจ้าของโพสเท่านั้น

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

## Lab 6: Bug Hunt

### Bug 1: Status ไม่ update ทุกคนขึ้น offline