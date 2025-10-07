// --- Config ---
const MODEL_NAME = "gemini-2.5-flash-image-preview";
const API_BASE_URL = "https://generativelanguage.googleapis.com/v1beta/models";
const apiKey = "YOUR_API_KEY_HERE"; // Add your Google Gemini API key here

// --- Elements ---
const imageUpload = document.getElementById('imageUpload');
const inputImage = document.getElementById('inputImage');
const previewPlaceholder = document.getElementById('previewPlaceholder');
const redesignButton = document.getElementById('redesignButton');
const buttonText = document.getElementById('buttonText');
const spinner = document.getElementById('spinner');
const messageBox = document.getElementById('messageBox');
const styleOptionsContainer = document.getElementById('styleOptions');
const originalDisplay = document.getElementById('originalDisplay');
const resultDisplay = document.getElementById('resultDisplay');

let base64Image = null;
let selectedStyle = 'Rich';

// --- Styles List ---
const styles = [
  { id: 'rich', name: 'Rich', description: 'Luxurious, opulent materials, and deep colors.' },
  { id: 'vintage', name: 'Vintage', description: 'Distressed items, warm lighting, and muted tones.' },
  { id: 'classic', name: 'Classic', description: 'Symmetrical, elegant traditional style.' },
  { id: 'modern', name: 'Modern', description: 'Minimalist, clean lines, and open space.' },
];

// --- Functions ---
function renderStyleOptions() {
  styleOptionsContainer.innerHTML = styles.map((s, i) => `
    <div class="style-option">
      <input type="radio" id="${s.id}" name="designStyle" value="${s.name}" class="hidden" ${i === 0 ? 'checked' : ''} onchange="updateSelectedStyle('${s.name}')">
      <label for="${s.id}" class="block p-3 border-2 border-indigo-200 rounded-lg text-center font-medium text-gray-700 hover:bg-indigo-50 transition-colors">
        ${s.name}
        <p class="text-xs text-gray-500 mt-1">${s.description}</p>
      </label>
    </div>
  `).join('');
}

function updateSelectedStyle(name) {
  selectedStyle = name;
}

function showMessage(msg, type) {
  messageBox.textContent = msg;
  messageBox.className = 'p-4 rounded-lg text-sm mb-6';
  messageBox.classList.remove('hidden');
  if (type === 'error') messageBox.classList.add('bg-red-100', 'text-red-700');
  else if (type === 'success') messageBox.classList.add('bg-green-100', 'text-green-700');
  else messageBox.classList.add('bg-blue-100', 'text-blue-700');
}

function hideMessage() {
  messageBox.classList.add('hidden');
}

function setLoadingState(isLoading) {
  redesignButton.disabled = isLoading || !base64Image;
  imageUpload.disabled = isLoading;
  if (isLoading) {
    buttonText.textContent = 'Redesigning...';
    spinner.classList.remove('hidden');
  } else {
    buttonText.textContent = 'Redesign Room';
    spinner.classList.add('hidden');
  }
}

function fileToBase64(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result.split(',')[1]);
    reader.onerror = err => reject(err);
    reader.readAsDataURL(file);
  });
}

function previewImage() {
  const file = imageUpload.files[0];
  base64Image = null;
  hideMessage();

  if (!file) {
    inputImage.classList.add('hidden');
    previewPlaceholder.classList.remove('hidden');
    redesignButton.disabled = true;
    return;
  }
  if (file.size > 1024 * 1024) {
    showMessage("The image file is too large. Please select an image under 1MB.", 'error');
    imageUpload.value = '';
    redesignButton.disabled = true;
    return;
  }

  const reader = new FileReader();
  reader.onload = async e => {
    inputImage.src = e.target.result;
    inputImage.classList.remove('hidden');
    previewPlaceholder.classList.add('hidden');

    originalDisplay.innerHTML = `<img class="max-h-full max-w-full object-contain rounded-md" src="${e.target.result}" alt="Original">`;
    resultDisplay.innerHTML = `<span class="text-gray-400">Redesigned image will appear here</span>`;

    try {
      base64Image = await fileToBase64(file);
      redesignButton.disabled = false;
    } catch {
      showMessage("Error processing image file.", 'error');
      redesignButton.disabled = true;
    }
  };
  reader.readAsDataURL(file);
}

async function fetchWithBackoff(payload, retry = 0) {
  const url = `${API_BASE_URL}/${MODEL_NAME}:generateContent?key=${apiKey}`;
  const MAX_RETRIES = 5;
  const INITIAL_BACKOFF_MS = 1000;

  try {
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });

    if (!res.ok) {
      if (res.status === 429 && retry < MAX_RETRIES) {
        const delay = INITIAL_BACKOFF_MS * 2 ** retry + Math.random() * 1000;
        await new Promise(r => setTimeout(r, delay));
        return fetchWithBackoff(payload, retry + 1);
      }
      const err = await res.json();
      throw new Error(err.error?.message || "API Error");
    }

    return res.json();
  } catch (err) {
    if (retry < MAX_RETRIES) {
      const delay = INITIAL_BACKOFF_MS * 2 ** retry + Math.random() * 1000;
      await new Promise(r => setTimeout(r, delay));
      return fetchWithBackoff(payload, retry + 1);
    }
    throw err;
  }
}

async function startRedesign() {
  if (!base64Image) return showMessage("Please upload a room image first.", 'error');

  setLoadingState(true);
  hideMessage();
  resultDisplay.innerHTML = `<span class="text-gray-400 animate-pulse">Waiting for AI magic...</span>`;

  const prompt = `Redesign the provided room image into a stunning ${selectedStyle} style. Keep layout, but change furniture, decor, and color tone. Output a realistic, high-quality result.`;

  const payload = {
    contents: [{
      parts: [
        { text: prompt },
        { inlineData: { mimeType: imageUpload.files[0].type, data: base64Image } }
      ]
    }],
    generationConfig: { responseModalities: ['IMAGE'] }
  };

  try {
    const result = await fetchWithBackoff(payload);
    const base64Data = result?.candidates?.[0]?.content?.parts?.find(p => p.inlineData)?.inlineData?.data;

    if (!base64Data) throw new Error("No image returned from AI");

    resultDisplay.innerHTML = `<img class="max-h-full max-w-full object-contain rounded-md" src="data:image/png;base64,${base64Data}" alt="Redesigned">`;
    showMessage(`Successfully transformed your room to a ${selectedStyle} style!`, 'success');
  } catch (err) {
    resultDisplay.innerHTML = `<span class="text-red-500 text-center p-4">Error: ${err.message}</span>`;
    showMessage(`Redesign failed: ${err.message}`, 'error');
  } finally {
    setLoadingState(false);
  }
}

// Init
window.onload = () => {
  renderStyleOptions();
  updateSelectedStyle(styles[0].name);
  setLoadingState(false);
};
