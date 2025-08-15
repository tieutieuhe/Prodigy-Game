<script setup>
// Import the functions you need from the SDKs you need
import { initializeApp } from "firebase/app";
import { getFirestore, collection, addDoc, getDocs, query, where } from "firebase/firestore";
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

const name = ref('')

/**
 * Kiểm tra xem username đã tồn tại trong bộ sưu tập 'users' hay chưa.
 * @param username Tên người dùng cần kiểm tra.
 * @returns true nếu username đã tồn tại, false nếu chưa.
 */
const checkUsernameExists = async (username) => {
  const db = getFirestore();
  const usersRef = collection(db, 'users');

  // Tạo một truy vấn để tìm tài liệu có trường 'name' (hoặc 'username') bằng với giá trị username
  const q = query(usersRef, where('name', '==', username));

  try {
    const querySnapshot = await getDocs(q);

    // Nếu querySnapshot.empty là false, có nghĩa là đã tìm thấy ít nhất một tài liệu khớp.
    // Điều đó có nghĩa là username đã tồn tại.
    return !querySnapshot.empty;
  } catch (error) {
    console.error("Lỗi khi kiểm tra username:", error);
    // Trong trường hợp lỗi, bạn có thể chọn trả về true (để ngăn đăng ký trùng)
    // hoặc false (nếu bạn muốn cho phép người dùng thử lại)
    // Ở đây tôi chọn false để báo lỗi và người dùng có thể thử lại.
    throw error; // Hoặc return false; tùy vào logic xử lý lỗi của bạn
  }
};

const loginOrRegister = async () => {
  if (!name.value) return
  const db = getFirestore();
  const usersCollectionRef = collection(db, 'users');

  // check if a user already exists
 if (await checkUsernameExists(name.value)) {
   console.log(
       'Exists !'
   )
  return
 }

  try {
    const newDocRef = await addDoc(usersCollectionRef, {
      name: name.value,
      timestamp: Date.now(), // Thường thêm timestamp khi tạo
      // Thêm các trường khác
    });

    console.log('Tài liệu người dùng mới đã được tạo với ID: ', newDocRef.id);
  } catch (error) {
    console.error('Lỗi khi thêm tài liệu: ', error);
  }
}


onMounted(()=>{
	// Initialize Firebase
	const app = initializeApp(firebaseConfig);
	const analytics = getAnalytics(app);
})
</script>
<template>

    <UInput v-model="name" placeholder="Tên của bạn" class="mr-4" />
    <UButton @click="loginOrRegister">Chơi ngay</UButton>
  
</template>