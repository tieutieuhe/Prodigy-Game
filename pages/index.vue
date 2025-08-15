<script setup>
// Import the functions you need from the SDKs you need
import { initializeApp } from "firebase/app";
import { getFirestore, doc, setDoc } from "firebase/firestore";
import { getAnalytics } from "firebase/analytics";
// TODO: Add SDKs for Firebase products that you want to use
// https://firebase.google.com/docs/web/setup#available-libraries

// Your web app's Firebase configuration
// For Firebase JS SDK v7.20.0 and later, measurementId is optional
const firebaseConfig = {
  apiKey: "AIzaSyAHQ2_QE8gKqbMcn-qCxpqzHpyChNSGOiU",
  authDomain: "prodigygame-1f9be.firebaseapp.com",
  projectId: "prodigygame-1f9be",
  storageBucket: "prodigygame-1f9be.firebasestorage.app",
  messagingSenderId: "608918487950",
  appId: "1:608918487950:web:5572523edefee2c53b9adf",
  measurementId: "G-NXNNKWF0VM"
};

const writeTestData = () => {
  // 1. Lấy thể hiện của Cloud Firestore
  const db = getFirestore();

  // 2. Định nghĩa đường dẫn tới tài liệu (document) bạn muốn ghi
  // Ở đây, tôi sẽ ghi vào collection 'users', với document ID là 'prodigy'
  const userDocRef = doc(db, 'users', 'prodigy');

  // 3. Ghi dữ liệu vào tài liệu sử dụng setDoc()
  // Nếu tài liệu chưa tồn tại, nó sẽ được tạo mới.
  // Nếu tài liệu đã tồn tại, nó sẽ được ghi đè hoàn toàn (mặc định)
  // hoặc được hợp nhất (merge) nếu bạn thêm { merge: true }.
  setDoc(userDocRef, {
    name: 'Prodigy',
    age: 20,
    timestamp: new Date() // Thêm một trường timestamp để dễ theo dõi
  })
      .then(() => {
        console.log("Dữ liệu đã được lưu vào Firestore thành công!");
      })
      .catch((error) => {
        console.error("Ghi dữ liệu vào Firestore thất bại:", error);
      });
};

onMounted(()=>{
	// Initialize Firebase
	const app = initializeApp(firebaseConfig);
	const analytics = getAnalytics(app);
  // const database = getDatabase();
  writeTestData();
})
</script>
<template>
  tét
</template>