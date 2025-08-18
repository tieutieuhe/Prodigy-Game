<script setup>
import { initializeApp, getApps, getApp } from "firebase/app";
import { getFirestore, collection, addDoc, getDocs, query, where, doc, setDoc, onSnapshot, orderBy, limit } from "firebase/firestore";

const runtimeConfig = useRuntimeConfig()

let db;
let leaderboardUnsubscribe = null;

onMounted(() => {
  if (!getApps().length) {
    initializeApp(runtimeConfig.public.firebase);
  }
  db = getFirestore(getApp());
  fetchLeaderboard();
});

// --- BIẾN TRẠNG THÁI ỨNG DỤNG ---
const view = ref('login');
const username = ref('');
const selectedClass = ref(1);
const leaderboardClass = ref(1);

const user = reactive({
  id: null,
  name: '',
  classLevel: 1,
  highScore: 0
});

const game = reactive({
  num1: 0,
  num2: 0,
  operator: '+',
  answer: '',
  correctAnswer: 0,
  score: 0,
  timer: 30,
  timerInterval: null,
  feedback: ''
});

const leaderboard = ref([]);
const answerOptions = ref([]);

// --- LOGIC XỬ LÝ NGƯỜI DÙNG ---
const start = async () => {
  if (!username.value.trim()) {
    console.error('Vui lòng nhập tên của bạn!');
    return;
  }

  const usersRef = collection(db, 'users');
  const q = query(usersRef, where("name", "==", username.value.trim()));

  const querySnapshot = await getDocs(q);
  if (querySnapshot.empty) {
    const newUserDoc = await addDoc(usersRef, {
      name: username.value.trim(),
      highScore: 0,
      classLevel: selectedClass.value,
      createdAt: new Date()
    });
    user.id = newUserDoc.id;
    user.name = username.value.trim();
    user.highScore = 0;
    user.classLevel = selectedClass.value;
  } else {
    const existingUser = querySnapshot.docs[0];
    const data = existingUser.data();
    user.id = existingUser.id;
    user.name = data.name;
    user.highScore = data.highScore;
    user.classLevel = selectedClass.value;

    const userDocRef = doc(db, 'users', user.id);
    await setDoc(userDocRef, { classLevel: selectedClass.value }, { merge: true });
  }

  startGame();
};

// --- LOGIC TRÒ CHƠI ---
const startGame = () => {
  game.score = 0;
  game.timer = 30;
  game.feedback = '';
  view.value = 'playing';
  generateProblem();

  if (game.timerInterval) clearInterval(game.timerInterval);

  game.timerInterval = setInterval(() => {
    game.timer--;
    if (game.timer <= 0) {
      endGame();
    }
  }, 1000);
};

const generateAnswerOptions = () => {
  const options = new Set();
  options.add(game.correctAnswer);

  while (options.size < 4) {
    let wrongAnswer;
    const deviation = Math.floor(Math.random() * 20) + 1;
    const isPositive = Math.random() > 0.5;

    if (isPositive) {
      wrongAnswer = game.correctAnswer + deviation;
    } else {
      wrongAnswer = game.correctAnswer - deviation;
    }

    if (wrongAnswer !== game.correctAnswer && wrongAnswer >= 0) {
      options.add(wrongAnswer);
    }
  }

  answerOptions.value = Array.from(options).sort(() => Math.random() - 0.5);
};

const generateProblem = () => {
  const operators = ['+', '-', '*', '/'];
  game.operator = operators[Math.floor(Math.random() * operators.length)];

  const level = user.classLevel;
  let maxNum1, maxNum2;

  switch (level) {
    case 1:
      maxNum1 = 20; maxNum2 = 20;
      game.operator = ['+', '-'][Math.floor(Math.random() * 2)];
      break;
    case 2:
      maxNum1 = 50; maxNum2 = 50;
      game.operator = ['+', '-', '*'][Math.floor(Math.random() * 3)];
      break;
    case 3:
      maxNum1 = 100; maxNum2 = 100;
      game.operator = operators[Math.floor(Math.random() * operators.length)];
      break;
    case 4:
      maxNum1 = 200; maxNum2 = 200;
      break;
    case 5:
      maxNum1 = 500; maxNum2 = 500;
      break;
    default:
      maxNum1 = 20; maxNum2 = 20;
      game.operator = ['+', '-'][Math.floor(Math.random() * 2)];
  }

  if (game.operator === '+') {
    game.num1 = Math.floor(Math.random() * maxNum1) + 1;
    game.num2 = Math.floor(Math.random() * maxNum2) + 1;
    game.correctAnswer = game.num1 + game.num2;
  } else if (game.operator === '-') {
    game.num1 = Math.floor(Math.random() * maxNum1) + maxNum2;
    game.num2 = Math.floor(Math.random() * game.num1) + 1;
    game.correctAnswer = game.num1 - game.num2;
  } else if (game.operator === '*') {
    game.num1 = Math.floor(Math.random() * 10) + 1;
    game.num2 = Math.floor(Math.random() * 10) + 1;
    if (level >= 3) {
      game.num1 = Math.floor(Math.random() * 20) + 1;
      game.num2 = Math.floor(Math.random() * 20) + 1;
    }
    game.correctAnswer = game.num1 * game.num2;
  } else if (game.operator === '/') {
    const divisor = Math.floor(Math.random() * (level * 5)) + 2;
    const result = Math.floor(Math.random() * (level * 5)) + 1;
    game.num1 = divisor * result;
    game.num2 = divisor;
    game.correctAnswer = result;
  }

  generateAnswerOptions();
};

const problemText = computed(() => {
  return `${game.num1} ${game.operator} ${game.num2} = ?`;
});

const submitAnswer = (selectedAnswer) => {
  if (selectedAnswer === game.correctAnswer) {
    game.score += 10;
    game.feedback = 'correct';
  } else {
    game.feedback = 'incorrect';
  }

  setTimeout(() => {
    game.feedback = '';
    generateProblem();
  }, 500);
};

const endGame = async () => {
  clearInterval(game.timerInterval);
  view.value = 'gameOver';

  if (game.score > user.highScore) {
    user.highScore = game.score;
    const userDocRef = doc(db, 'users', user.id);
    await setDoc(userDocRef, { highScore: game.score }, { merge: true });
  }
};

// --- LOGIC BẢNG XẾP HẠNG ---
const fetchLeaderboard = () => {
  if (leaderboardUnsubscribe) {
    leaderboardUnsubscribe();
  }
  const usersRef = collection(db, 'users');
  const q = query(
      usersRef,
      // Đảm bảo giá trị của leaderboardClass là Number khi truy vấn
      where("classLevel", "==", Number(leaderboardClass.value)),
      orderBy("highScore", "desc"),
      limit(10)
  );

  leaderboardUnsubscribe = onSnapshot(q, (querySnapshot) => {
    const topUsers = [];
    querySnapshot.forEach((doc) => {
      topUsers.push({ id: doc.id, ...doc.data() });
    });
    leaderboard.value = topUsers;
  });
};

const showLeaderboard = () => {
  view.value = 'leaderboard';
  if (user.classLevel) {
    // Đảm bảo giá trị được gán là Number
    leaderboardClass.value = Number(user.classLevel);
  } else {
    leaderboardClass.value = 1;
  }
};

const backToLogin = () => {
  view.value = 'login';
  username.value = '';
};

// Theo dõi sự thay đổi của leaderboardClass để cập nhật bảng xếp hạng
watch(leaderboardClass, (newVal) => {
  fetchLeaderboard();
});

</script>

<template>
  <div class="bg-blue-50 min-h-screen flex items-center justify-center font-sans p-4">
    <UCard class="w-full max-w-md shadow-lg">
      <template #header>
        <h1 class="text-2xl font-bold text-center text-blue-700">🧠 Bé Vui Học Toán 🧠</h1>
      </template>

      <div v-if="view === 'login'" class="space-y-4">
        <p class="text-center text-gray-600">Hãy nhập tên của bạn và chọn lớp để bắt đầu!</p>
        <UInput v-model="username" size="xl" placeholder="Tên của bé là..." @keyup.enter="start" />

        <div class="flex items-center space-x-2">
          <span class="text-gray-700">Chọn lớp:</span>
          <USelectMenu v-model="selectedClass" :items="[1, 2, 3, 4, 5]" class="w-24" />
        </div>

        <div class="flex justify-center space-x-4">
          <UButton size="xl" @click="start">Bắt đầu chơi!</UButton>
          <UButton size="xl" color="gray" @click="showLeaderboard">Bảng xếp hạng</UButton>
        </div>
      </div>

      <div v-if="view === 'playing'" class="space-y-4">
        <div class="flex justify-between items-center text-lg">
          <span class="font-semibold text-green-600">Điểm: {{ game.score }}</span>
          <span class="font-bold text-red-500">⏰ {{ game.timer }}</span>
        </div>
        <div class="bg-blue-100 p-6 rounded-lg text-center">
          <p class="text-4xl font-bold text-gray-800 tracking-widest">{{ problemText }}</p>
        </div>

        <div class="grid grid-cols-2 gap-4">
          <UButton
              v-for="answerOption in answerOptions"
              :key="answerOption"
              size="xl"
              block
              class="text-2xl font-bold py-6"
              @click="submitAnswer(answerOption)">
            {{ answerOption }}
          </UButton>
        </div>

        <div v-if="game.feedback" class="fixed inset-0 flex items-center justify-center rounded-md pointer-events-none"
             :class="{ 'bg-green-500/80': game.feedback === 'correct', 'bg-red-500/80': game.feedback === 'incorrect' }">
          <p class="text-white text-3xl font-bold">{{ game.feedback === 'correct' ? 'Đúng rồi!' : 'Sai rồi!' }}</p>
        </div>
      </div>

      <div v-if="view === 'gameOver'" class="text-center space-y-4">
        <h2 class="text-3xl font-bold text-blue-800">Hết giờ!</h2>
        <p class="text-xl text-gray-700">Điểm cuối cùng của bạn là:</p>
        <p class="text-5xl font-extrabold text-green-600 animate-pulse">{{ game.score }}</p>
        <p class="text-lg">Điểm cao nhất: {{ user.highScore }}</p>
        <div class="flex justify-center space-x-4 pt-4">
          <UButton size="xl" @click="startGame">Chơi lại</UButton>
          <UButton size="xl" color="gray" @click="showLeaderboard">Bảng xếp hạng</UButton>
        </div>
      </div>

      <div v-if="view === 'leaderboard'" class="space-y-4">
        <h2 class="text-2xl font-bold text-center text-yellow-500">🏆 Bảng Vàng 🏆</h2>

        <div class="flex items-center justify-center space-x-2">
          <span class="text-gray-700">Lớp:</span>
          <USelectMenu v-model="leaderboardClass" :items="[1, 2, 3, 4, 5]" class="w-24" />
        </div>

        <div v-if="leaderboard.length === 0" class="text-center text-gray-500">
          Chưa có ai trên bảng xếp hạng của lớp này. Hãy là người đầu tiên!
        </div>
        <ul v-else class="space-y-2">
          <li v-for="(player, index) in leaderboard" :key="player.id" class="flex items-center justify-between p-3 rounded-lg"
              :class="{
              'bg-yellow-100': index === 0,
              'bg-gray-200': index === 1,
              'bg-orange-100': index === 2,
              'bg-gray-50': index > 2
            }">
            <span class="font-semibold text-lg">
              <span v-if="index === 0">🥇</span>
              <span v-else-if="index === 1">🥈</span>
              <span v-else-if="index === 2">🥉</span>
              <span v-else class="inline-block w-6 text-center">{{ index + 1 }}.</span>
              {{ player.name }}
            </span>
            <span class="font-bold text-blue-600">{{ player.highScore }} điểm</span>
          </li>
        </ul>
        <UButton v-if="user.name" size="lg" color="gray" block @click="startGame">Chơi với tên {{ user.name }}</UButton>
        <UButton size="lg" color="gray" block @click="backToLogin">Quay lại</UButton>
      </div>
    </UCard>
  </div>
</template>

<style>
/* Thêm một vài style global nếu cần */
body {
  font-family: 'Inter', sans-serif;
}
</style>