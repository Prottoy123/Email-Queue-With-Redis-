# 🚀 Redis Message Queue (Background Job Processing)

This repository demonstrates a First-Principles implementation of a decoupled Message Queue using pure Node.js and Redis. This architecture prevents main server blocking during heavy API operations (e.g., sending emails, processing payments) by offloading tasks to background workers.

---

## 🧠 Core Concept: First-Principles Architecture

যেকোনো Message Queue-এর মূলে থাকে **Decoupling** (মেইন সার্ভার থেকে ভারী কাজ আলাদা করে ফেলা) এবং **FIFO** (First-In, First-Out) মেকানিজম। আমাদের সিস্টেমটি তিনটি প্রধান অংশে বিভক্ত:

1. **The Producer (মেইন API):** ইউজারের রিকোয়েস্ট গ্রহণ করে এবং ভারী কাজটির ডেটা একটি Redis List-এ ফেলে রেখে ইউজারকে সাথে সাথে রেসপন্স দিয়ে দেয়। 
2. **The Queue (Redis List):** ডেটাগুলো ক্রমানুসারে (সিরিয়ালে) জমা থাকার জায়গা।
3. **The Consumer / Worker (ব্যাকগ্রাউন্ড প্রসেস):** সম্পূর্ণ আলাদা একটি ফাইল, যা ব্যাকগ্রাউন্ডে চলতে থাকে। এটি ডেটাবেজ থেকে একটি একটি করে কাজ টেনে নেয় এবং প্রসেস করে।

---

## 🛠️ Redis Commands Master Sheet

এই আর্কিটেকচারটি মূলত দুটি ম্যাজিক কমান্ডের ওপর দাঁড়িয়ে আছে:

| Command | Action | Explanation |
| :--- | :--- | :--- |
| `lpush` | **Left Push** | ডেটাকে লিস্টের বাঁ দিকে (শুরুতে) ইনসার্ট করে। Producer এটি ব্যবহার করে দ্রুত কাজ জমা দেওয়ার জন্য। |
| `brpop` | **Blocking Right Pop** | লিস্টের ডান দিক (শেষে) থেকে ডেটা টেনে বের করে এবং মুছে ফেলে। লিস্ট খালি থাকলে এটি ক্র্যাশ করে না, বরং নতুন ডেটা আসা পর্যন্ত **অপেক্ষা (Block)** করে। Worker এটি ব্যবহার করে। |

---

## 🏥 Real-World Scenario (Health Bridge Email System)

**সমস্যা:** ৫ জন রোগী একসাথে অ্যাপয়েন্টমেন্ট বুক করেছে। মেইন API থেকে ৫ জনকে ইমেইল পাঠাতে ৫ সেকেন্ড সময় লাগবে, ফলে ইউজারের ব্রাউজার ৫ সেকেন্ড লোডিং স্ক্রিনে আটকে থাকবে।
**সমাধান:** মেইন API ৫টি ইমেইলের ডেটা `lpush` করে লিস্টে ফেলে রাখবে (সময় লাগবে ০.১ সেকেন্ড)। ব্যাকগ্রাউন্ড Worker `brpop` দিয়ে একটি একটি করে ইমেইল পাঠাতে থাকবে।

---

## 💻 Code Implementation

প্রোডাকশন-স্ট্যান্ডার্ড সিস্টেমের জন্য আমাদের কোড দুটি আলাদা ফাইলে থাকবে।

### ১. `server.js` (The Producer)
এই ফাইলটি ইউজারের রিকোয়েস্ট হ্যান্ডেল করবে। 

```javascript
import express from "express";
import Redis from "ioredis";

const app = express();
const redis = new Redis();
const QUEUE_KEY = "queue:emails";

app.post("/book-appointment", async (req, res) => {
  const { patients } = req.body; // Array of emails

  // ডেটাগুলো Redis List-এ ঢুকিয়ে দেওয়া হলো (lpush)
  for (const email of patients) {
    const job = { to: email, subject: "Appointment Confirmed!" };
    await redis.lpush(QUEUE_KEY, JSON.stringify(job));
  }

  // ইমেইল সেন্ড হওয়ার জন্য অপেক্ষা না করেই ইউজারকে রেসপন্স!
  res.status(200).json({ message: "Bookings confirmed! Emails are queued." });
});
app.listen(3000, () => console.log("Server is running on port 3000"));
```

---

### ২. `worker.js` (The Consumer)
এই ফাইলটি সম্পূর্ণ আলাদা টার্মিনালে রান করবে (`node worker.js`)। এটি মেইন সার্ভারকে ডিস্টার্ব না করে নিজের মতো কাজ করবে।

```javascript
import Redis from "ioredis";
const redis = new Redis();
const QUEUE_KEY = "queue:emails";

async function startWorker() {
  console.log("Background Worker running! Waiting for jobs...");

  // ইনফিনিট লুপ - এটি সার্বক্ষণিক চলতে থাকবে
  while (true) {
    // brpop লিস্টে ডেটা না আসা পর্যন্ত অপেক্ষা করবে (0 মানে আনলিমিটেড সময়)
    // যখনই lpush হবে, এটি ছোঁ মেরে ডান দিকের ডেটাটা নিয়ে নেবে
    const result = await redis.brpop(QUEUE_KEY, 0); 
    
    // result[1] এ আসল JSON স্ট্রিং ডেটা থাকে
    const job = JSON.parse(result[1]);

    console.log(`Processing email for: ${job.to}...`);

    // ইমেইল পাঠানোর ফেক ডিলে (১ সেকেন্ড)
    await new Promise((resolve) => setTimeout(resolve, 1000));

    console.log(`✅ Email sent to ${job.to}`);
  }
}

startWorker();


