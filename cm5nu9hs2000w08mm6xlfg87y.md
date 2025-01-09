---
title: "Comprehensive Guide: Encrypting Images, Uploading to Cloudinary, and Serving Them Securely"
seoTitle: "Encrypting Images, Uploading to Cloudinary, and Serving Them Securely"
datePublished: Wed Jan 08 2025 11:51:15 GMT+0000 (Coordinated Universal Time)
cuid: cm5nu9hs2000w08mm6xlfg87y
slug: comprehensive-guide-encrypting-images-uploading-to-cloudinary-and-serving-them-securely
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1736336845591/69561304-6153-4593-af17-8b70ed9e0c23.png
tags: web-development, nodejs, encryption, cloudinary, multer

---

This guide explains how to implement a secure system for encrypting images, uploading them to a cloud storage solution ([Cloudinary](https://cloudinary.com/)), and serving them when needed. We’ll walk through each step in detail, from setting up key generation to encrypting and decrypting images, uploading to Cloudinary, and securely serving them. By the end, you'll understand how everything ties together to form a robust, secure image handling process.

---

## **Step 1: Key Generation**

To secure the encryption and decryption process, we'll use [RSA](https://www.geeksforgeeks.org/rsa-algorithm-cryptography/) (asymmetric cryptography) to encrypt the AES (symmetric) key used for image encryption.

### **Generating RSA Keys**

```javascript
import crypto from "crypto";
import fs from "fs";

// Generate RSA key pair
const { publicKey, privateKey } = crypto.generateKeyPairSync("rsa", {
  modulusLength: 2048, // Key size in bits
  publicKeyEncoding: {
    type: "spki", // Recommended format for public key
    format: "pem",
  },
  privateKeyEncoding: {
    type: "pkcs8", // Recommended format for private key
    format: "pem",
  },
});

// Save public key to a file
fs.writeFileSync("public_key.pem", publicKey);
console.log("Public key saved to public_key.pem");

// Save private key to a file
fs.writeFileSync("private_key.pem", privateKey);
console.log("Private key saved to private_key.pem");
```

This generates an RSA key pair and saves the keys to files for use in encryption and decryption.

---

## **Step 2: Setting Up Image Uploads**

We’ll use [`multer`](https://github.com/expressjs/multer#readme), a middleware for handling file uploads in Node.js, to handle user-uploaded images.

### **Installing Multer**

```bash
npm install multer
```

### **Configuring Multer**

```javascript
import multer from "multer";
import path from "path";

const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, "uploads/"); // Directory to save uploaded files
  },
  filename: (req, file, cb) => {
    const uniqueSuffix = Date.now() + "-" + Math.round(Math.random() * 1E9);
    cb(null, `${uniqueSuffix}-${file.originalname}`);
  },
});

export const upload = multer({ storage });
```

This code configures Multer to save uploaded images to a directory called `uploads/`.

---

## **Step 3: Encrypting Images**

We'll use [**AES-GCM**](https://www.sharesecure.link/articles/the-ultimate-developers-guide-to-aes-gcm-encryption-with-web-cryptography-api#understanding-encryption-how-it-all-works) (a symmetric encryption algorithm) to encrypt the image data and RSA to securely store the encryption key.

### **Encryption Function**

```javascript
import crypto from "crypto";
import fs from "fs";

// Load RSA public key
const publicKey = fs.readFileSync("public_key.pem", "utf8");

export const encryptImage = (imageFile) => {
  const algorithm = "aes-256-gcm";
  const key = crypto.randomBytes(32); // AES key
  const iv = crypto.randomBytes(16); // Initialization vector
  const cipher = crypto.createCipheriv(algorithm, key, iv);

  // Read the image file into a buffer
  const imageBuffer = fs.readFileSync(imageFile.path);

  let encrypted = cipher.update(imageBuffer);
  encrypted = Buffer.concat([encrypted, cipher.final()]);
  const authTag = cipher.getAuthTag();

  // Encrypt the AES key with RSA public key
  const encryptedKey = crypto.publicEncrypt(publicKey, key);

  return {
    iv: iv.toString("hex"),
    encryptedData: encrypted.toString("base64"),
    authTag: authTag.toString("hex"),
    encryptedKey: encryptedKey.toString("base64"),
  };
};
```

This function encrypts the image file using AES-GCM and encrypts the AES key with the RSA public key.

---

## **Step 4: Uploading to Cloudinary**

### **Installing Cloudinary SDK**

```bash
npm install cloudinary
```

### **Configuring Cloudinary**

```javascript
import { v2 as cloudinary } from "cloudinary";

cloudinary.config({
  cloud_name: "your-cloud-name", // Replace with your Cloudinary cloud name
  api_key: "your-api-key",       // Replace with your Cloudinary API key
  api_secret: "your-api-secret", // Replace with your Cloudinary API secret
});
```

### **Uploading Encrypted Images**

```javascript
import fs from "fs";

export const uploadToCloudinary = async (encryptedData, fileName) => {
  const tempFilePath = `temp/${fileName}.json`;
  fs.writeFileSync(tempFilePath, JSON.stringify(encryptedData));

  const result = await cloudinary.uploader.upload(tempFilePath, {
    resource_type: "raw",
    folder: "encrypted_images",
  });

  fs.unlinkSync(tempFilePath); // Delete temporary file
  return result.secure_url; // Return the uploaded file URL
};
```

This uploads the encrypted image data as a JSON file to Cloudinary.

---

## **Step 5: Decrypting and Serving Images**

### **Decryption Function**

```javascript
// Load RSA private key
const privateKey = fs.readFileSync("private_key.pem", "utf8");

export const decryptImage = (encryptedData) => {
  const { encryptedData: data, encryptedKey, iv, authTag } = encryptedData;

  // Decrypt the AES key using RSA private key
  const key = crypto.privateDecrypt(privateKey, Buffer.from(encryptedKey, "base64"));

  const decipher = crypto.createDecipheriv("aes-256-gcm", key, Buffer.from(iv, "hex"));
  decipher.setAuthTag(Buffer.from(authTag, "hex"));

  let decrypted = decipher.update(Buffer.from(data, "base64"));
  decrypted = Buffer.concat([decrypted, decipher.final()]);

  return decrypted;
};
```

This function decrypts the AES-encrypted image using the RSA private key and the provided metadata.

### **Serving Decrypted Images**

```javascript
import axios from "axios";

export const decodeUserImageController = async (req, res) => {
  const { version, public_id } = req.params;
  if (!public_id) {
    res.status(400).send("Invalid request: public_id is required");
    return;
  }

  // Fetch encrypted data from Cloudinary
  const encryptedData = (await axios.get(`https://res.cloudinary.com/your-cloud-name/raw/upload/v${version}/${public_id}.json`)).data;

  // Decrypt the image
  const decrypted = decryptImage(encryptedData);

  // Serve the decrypted image
  res.type("image/png").send(decrypted);
};
```

This code fetches the encrypted image metadata from Cloudinary, decrypts it, and serves it as a response.

---

## **Conclusion**

By following this guide, you can implement a secure system for encrypting and storing sensitive images. Using a combination of AES-GCM for encryption and RSA for key security ensures strong protection. Cloudinary provides a scalable storage solution for encrypted data, and the ability to decrypt and serve images on demand makes this system both flexible and secure.