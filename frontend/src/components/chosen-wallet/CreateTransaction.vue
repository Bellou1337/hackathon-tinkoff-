<script setup>
import { ref, onMounted } from 'vue';
import { getCookie } from '@/utils/cookies';
import { useRoute, useRouter } from 'vue-router';
import apiClient from '@/services';
import { QrcodeStream } from 'vue3-qrcode-reader';

// Reactive references
const title = ref('');
const amount = ref('');
const date = ref('');
const fiscalNumber = ref('');
const checkNumber = ref('');
const fiscalSign = ref('');
const documentType = ref('');

const dropdownOpen = ref(false);
const categories = ref([]);
const route = useRoute();
const router = useRouter();
const id = route.query.id;
const selectedOption = ref('');
const isScanning = ref(false); // Controls scanning state

// Fetch categories from API
const fetchCategories = async () => {
try {
const token = await getCookie('auth_token');
if (!token) throw new Error('Token not found');

const response = await apiClient.get('/category/get_all', {
headers: { Authorization: `Bearer ${token}` },
});

if (response.status === 200) {
categories.value = response.data.map(category => ({
name: category.name,
income: category.is_income,
active: false,
id: category.id,
}));
}
} catch (err) {
console.log(err);
}
};

// Create a new transaction
const createTransaction = async () => {
try {
const token = await getCookie('auth_token');
if (!token) throw new Error('Token not found');

const response = await apiClient.post(
'/transaction/add',
{
title: title.value,
category_id: selectedOption.value,
amount: Math.abs(amount.value),
wallet_id: id,
date: date.value,
// Include other necessary fields if required
},
{
headers: { Authorization: `Bearer ${token}` },
}
);

if (response.status === 200) {
router.push(`/profile/wallet?id=${id}`);
}
} catch (err) {
console.log(err);
}
};

onMounted(async () => {
await fetchCategories();
});

// Select a category
const selectCategory = (categoryId) => {
	selectedOption.value = categoryId;
	categories.value.forEach(category => {
	category.active = category.id === categoryId;
	});
	};

	// QR Scanner Handlers
	const onDecode = (decodedString) => {
	console.log('QR Code Decoded:', decodedString);

	// Parse the decoded string as URL parameters
	const params = new URLSearchParams(decodedString);

	// Extract parameters
	const t = params.get("t"); // Date
	const s = params.get("s"); // Amount
	const fn = params.get("fn"); // Fiscal Number
	const i = params.get("i"); // Check Number
	const fp = params.get("fp"); // Fiscal Sign
	const n = params.get("n"); // Document Type

	// Format the date from "yyyyMMddTHHmm" to "yyyy-MM-dd"
	if (t) {
	const datePart = t.split('T')[0]; // "20241123"
	const formattedDate = `${datePart.substring(0, 4)}-${datePart.substring(4, 6)}-${datePart.substring(6, 8)}`;
	date.value = formattedDate; // e.g., "2024-11-23"
	}

	// Assign other values
	amount.value = s || '';
	fiscalNumber.value = fn || '';
	checkNumber.value = i || '';
	fiscalSign.value = fp || '';
	documentType.value = n || '';

	stopScanning()
};

const onDecodeError = (error) => {
console.error('QR Code Decode Error:', error);
};

const startScanning = () => {
  if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
    navigator.mediaDevices.getUserMedia({ video: true })
      .then((stream) => {
        console.log("Доступ к камере получен");
        isScanning.value = true;  // Открытие потока видео
      })
      .catch((error) => {
        console.error("Ошибка доступа к камере:", error);
      });
  } else {
    console.error("Ваш браузер не поддерживает доступ к камере");
  }
};

const stopScanning = () => {
isScanning.value = false;
};
</script>

<template>
<div class="bg-slight-gray min-h-[calc(100vh-80px)] p-5 flex items-center justify-center">
<div class="bg-white shadow rounded p-16">
<p class="text-3xl font-bold text-slight-black text-center mb-8">Новая транзакция</p>
<div class="flex sm:flex-row flex-col gap-10 mb-4">
<div class="flex flex-col">
<!— QR Scanner Section —>
<div class="mb-8">
<button
@click="dropdownOpen = !dropdownOpen"
class="inline-block rounded-lg w-full bg-yellow-300 px-5 py-3 text-sm font-medium transition hover:bg-yellow-400"
>
Выберите категории
</button>
<div
v-show="dropdownOpen"
class="absolute mt-2 w-56 bg-white border rounded-lg shadow-lg z-10 p-2"
>
<label
v-for="(category, index) in categories"
:key="index"
class="flex items-center space-x-2"
>
<input
type="checkbox"
v-model="category.active"
:value="category.name"
:checked="selectedOption === category.id"
@change="selectCategory(category.id)"
class="h-4 w-4 text-green-500 border-gray-300 rounded focus:ring-green-500"
/>
<span>{{ category.name }}</span>
</label>
</div>
</div>

<!— QR Scanner Controls —>
<div class="mb-8">
<button
@click="startScanning"
class="inline-block rounded-lg w-full bg-blue-300 px-5 py-3 text-sm font-medium transition hover:bg-blue-400 mb-2"
:disabled="isScanning"
>
Начать сканирование
</button>
<button
@click="stopScanning"
class="inline-block rounded-lg w-full bg-red-300 px-5 py-3 text-sm font-medium transition hover:bg-red-400"
:disabled="!isScanning"
>
Закончить сканирование
</button>
</div>

<!— QR Scanner Component —>
<div v-if="isScanning" class="mb-8">
<qrcode-stream @decode="onDecode" @decode-error="onDecodeError" />
</div>

<!— Transaction Form Fields —>
<div class="flex flex-col gap-4">
<div class="relative">
<input
v-model="title"
type="text"
class="w-full rounded-lg border-gray-200 bg-gray-100 p-4 pe-12 text-sm shadow-sm transition hover:bg-gray-200"
placeholder="Название транзакции"
maxlength="20"
required
/>
</div>

<div class="relative">
<input
v-model="amount"
type="number"
class="w-full rounded-lg border-gray-200 bg-gray-100 p-4 pe-12 text-sm shadow-sm transition hover:bg-gray-200"
placeholder="Сумма"
required
style="-webkit-appearance: none; -moz-appearance: textfield; appearance: none"
/>
</div>


</div>
</div>

<div
class="bg-gray-200 p-5 my-10 h-32 rounded shadow">
<p class="text-lg font-bold mb-2 text-center">Дата</p>
<input v-model="date" class="rounded text-center" type="date" />
</div>
</div>

<div class="flex">
<button
@click="createTransaction"
class="inline-block rounded-lg w-full bg-yellow-300 px-5 py-3 text-sm font-medium transition hover:bg-yellow-400"
>
Создать
</button>
</div>
</div>
</div>
</template>

<style>
input[type='number']::-webkit-inner-spin-button,
input[type='number']::-webkit-outer-spin-button {
-webkit-appearance: none;
margin: 0;
}

input[type='number'] {
-moz-appearance: textfield;
appearance: textfield;
}
</style>