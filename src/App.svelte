<script>
	import { PDFDocument, rgb } from 'pdf-lib';
	import download from 'downloadjs';
	import { onMount } from 'svelte';

	let currentPDFData = null;
	let currentPDF = null;
	let pdfDoc = null;
	let currentFileName = '';
	let mergeFiles = [];
	let showOverlay = true; // Variable to control overlay visibility

	function closeDialog(dialogId) {
		document.getElementById(dialogId).style.display = 'none';
	}

	async function loadPDF(data, filename) {
		currentPDFData = data;
		currentFileName = filename;
		try {
			const blob = new Blob([data], { type: 'application/pdf' });
			const url = URL.createObjectURL(blob);

			pdfDoc = await pdfjsLib.getDocument(url).promise;
			currentPDF = await PDFDocument.load(data);

			document.getElementById('viewer').innerHTML = '';
			const objectTag = document.createElement('object');
			objectTag.data = url;
			objectTag.type = 'application/pdf';
			objectTag.width = '100%';
			objectTag.height = '100%';
			document.getElementById('viewer').appendChild(objectTag);

			showOverlay = false; // Hide overlay after loading PDF
		} catch (error) {
			console.error('Error loading PDF:', error);
			alert('Error loading PDF');
		}
	}

	function handleFileInputChange(event) {
		const file = event.target.files[0];
		if (file) {
			const reader = new FileReader();
			reader.onload = (e) => {
				loadPDF(e.target.result, file.name);
			};
			reader.readAsArrayBuffer(file);
		}
	}

	function handleOpenNewClick() {
		document.getElementById('fileInput').click();
	}
	function parsePageRanges(pageInput, numPages) {
	  const pages = new Set();
	  const ranges = pageInput.split(',');
  
	  for (const range of ranges) {
		const parts = range.trim().split('-');
		if (parts.length === 1) {
		  const page = parseInt(parts[0]);
		  if (!isNaN(page) && page >= 1 && page <= numPages) {
			pages.add(page);
		  }
		} else if (parts.length === 2) {
		  const start = parseInt(parts[0]);
		  const end = parseInt(parts[1]);
		  if (!isNaN(start) && !isNaN(end) && start <= end && start >= 1 && end <= numPages) {
			for (let i = start; i <= end; i++) {
			  pages.add(i);
			}
		  }
		}
	  }
	  return Array.from(pages);
	}
  
	async function handlePdfToImageClick() {
	  if (!currentPDF) {
		alert("Please load a PDF file first.");
		return;
	  }
	  document.getElementById('pdf2png-dialog').style.display = 'block';
	}
  
	async function handlePngConvertClick() {
	  const pagesOption = document.querySelector('input[name="png-pages"]:checked').value;
	  let pagesToConvert = [];
  
	  if (pagesOption === 'all') {
		pagesToConvert = Array.from({ length: pdfDoc.numPages }, (_, i) => i + 1);
	  } else {
		const pageInput = document.getElementById('png-page-input').value;
		pagesToConvert = parsePageRanges(pageInput, pdfDoc.numPages);
	  }
  
	  const baseName = currentFileName.replace('.pdf', '');
  
	  for (const pageNum of pagesToConvert) {
		try {
		  const page = await pdfDoc.getPage(pageNum);
		  const scale = 3;
		  const viewport = page.getViewport({ scale: scale });
		  const canvas = document.createElement('canvas');
		  const context = canvas.getContext('2d');
		  canvas.height = viewport.height;
		  canvas.width = viewport.width;
  
		  await page.render({ canvasContext: context, viewport: viewport }).promise;
  
		  canvas.toBlob((blob) => {
			const fileName = `${baseName} ${pageNum}.png`;
			download(blob, fileName, "image/png");
		  }, 'image/png', 1);
		} catch (error) {
		  console.error("Error converting to PNG:", error);
		  alert(`Error converting page ${pageNum} to PNG.`);
		}
	  }
  
	  closeDialog('pdf2png-dialog');
	}
  
	function handleSplitPdfClick() {
	  if (!currentPDF) {
		alert("Please load a PDF file first.");
		return;
	  }
	  document.getElementById('split-dialog').style.display = 'block';
	}
  
	async function handleSplitConfirmClick() {
	  const splitPage = parseInt(document.getElementById('split-page-input').value);
	  if (isNaN(splitPage) || splitPage < 2 || splitPage > currentPDF.getPageCount()) {
		alert("Please enter a valid page number.");
		return;
	  }
  
	  try {
		const baseName = currentFileName.replace('.pdf', '');
  
		const pdfDoc1 = await PDFDocument.create();
		const pdfDoc2 = await PDFDocument.create();
  
		const copiedPages1 = await pdfDoc1.copyPages(currentPDF, Array.from({ length: splitPage - 1 }, (_, i) => i));
		copiedPages1.forEach((page) => pdfDoc1.addPage(page));
  
		const copiedPages2 = await pdfDoc2.copyPages(currentPDF, Array.from({ length: currentPDF.getPageCount() - splitPage + 1 }, (_, i) => i + splitPage - 1));
		copiedPages2.forEach((page) => pdfDoc2.addPage(page));
  
		const pdfBytes1 = await pdfDoc1.save();
		const pdfBytes2 = await pdfDoc2.save();
  
		download(new Blob([pdfBytes1]), `${baseName} part 1.pdf`, "application/pdf");
		download(new Blob([pdfBytes2]), `${baseName} part 2.pdf`, "application/pdf");
	  } catch (error) {
		console.error("Error splitting PDF:", error);
		alert("Error splitting PDF");
	  }
  
	  closeDialog('split-dialog');
	}
  
	function handleMergePdfClick() {
	  if (!currentPDF) {
		alert("Please load a PDF file first.");
		return;
	  }
	  const mergeDialog = document.getElementById('merge-dialog');
	  mergeDialog.style.display = 'block';
  
	  mergeFiles.length = 0;
	  mergeFiles.push(new File([currentPDFData], currentFileName, { type: 'application/pdf' }));
	  updateMergeFilesList();
	}
  
	function handleAddMergeFileClick() {
	  document.getElementById('merge-file-input').click();
	}
  
	function handleMergeFileInputChange(event) {
	  const files = Array.from(event.target.files);
	  mergeFiles.push(...files);
	  updateMergeFilesList();
	}
  
	function updateMergeFilesList() {
	  const list = document.getElementById('merge-files-list');
	  list.innerHTML = mergeFiles.map((file, index) => `
		<div class="merge-file-item">
		  <span>${file.name}</span>
		  <button class="remove-file" data-index="${index}">X</button>
		</div>
	  `).join('');
  
	  document.querySelectorAll('.remove-file').forEach(button => {
		button.addEventListener('click', (e) => {
		  const index = parseInt(e.target.getAttribute('data-index'));
		  mergeFiles.splice(index, 1);
		  updateMergeFilesList();
		});
	  });
	}
  
	async function handleMergeButtonClick() {
	  if (mergeFiles.length < 2) {
		alert("Please select at least two PDF files to merge.");
		return;
	  }
  
	  try {
		const mergedPdf = await PDFDocument.create();
  
		for (const file of mergeFiles) {
		  const pdfBytes = await file.arrayBuffer();
		  const pdf = await PDFDocument.load(pdfBytes);
		  const copiedPages = await mergedPdf.copyPages(pdf, pdf.getPageIndices());
		  copiedPages.forEach((page) => mergedPdf.addPage(page));
		}
  
		const mergedPdfFile = await mergedPdf.save();
		download(new Blob([mergedPdfFile]), "merged.pdf", "application/pdf");
  
		mergeFiles.length = 0;
		updateMergeFilesList();
	  } catch (error) {
		console.error("Error merging PDFs:", error);
		alert("Error merging PDFs");
	  }
  
	  closeDialog('merge-dialog');
	}
  
	function handleCompressPdfClick() {
	  if (!currentPDF) {
		alert("Please load a PDF file first.");
		return;
	  }
	  document.getElementById('compress-dialog').style.display = 'block';
	}
  
	async function handleCompressConfirmClick() {
	  const compressionType = document.querySelector('input[name="compression-type"]:checked').value;
	  const progressBar = document.getElementById('compression-progress-bar');
	  const progressText = document.getElementById('compression-progress-text');
	  const progressDiv = document.getElementById('compression-progress');
  
	  progressDiv.style.display = 'block';
  
	  try {
		const pdf = await pdfjsLib.getDocument(URL.createObjectURL(new Blob([currentPDFData]))).promise;
		const allPages = [];
  
		for (let i = 1; i <= pdf.numPages; i++) {
		  const page = await pdf.getPage(i);
		  const viewport = page.getViewport({ scale: 2 });
		  const canvas = document.createElement('canvas');
		  const ctx = canvas.getContext('2d');
		  canvas.height = viewport.height;
		  canvas.width = viewport.width;
		  const renderContext = {
			canvasContext: ctx,
			viewport: viewport,
		  };
  
		  await page.render(renderContext).promise;
  
		  let pdfAsBase64 = null;
  
		  if (compressionType === 'size') {
			const desiredSizeKB = parseInt(document.getElementById('target-size').value);
			pdfAsBase64 = await compressImageToSizeDataUri(
			  canvas.toDataURL('image/jpeg', 1),
			  (desiredSizeKB / pdf.numPages) * 1024,
			  0.85
			);
		  } else {
			const compressionQuality = parseFloat(document.querySelector('input[name="compression-quality"]:checked').value);
			pdfAsBase64 = canvas.toDataURL('image/jpeg', Math.max(compressionQuality, 0.85));
		  }
  
		  allPages.push({
			data: pdfAsBase64,
			height: viewport.height / 2,
			width: viewport.width / 2,
		  });
  
		  const progress = Math.round((i / pdf.numPages) * 100);
		  progressBar.value = progress;
		  progressText.textContent = `${progress}%`;
		}
  
		const newPdf = await PDFDocument.create();
		for (let i = 0; i < allPages.length; i++) {
		  const img = await newPdf.embedJpg(allPages[i].data);
		  const page = newPdf.addPage([allPages[i].width, allPages[i].height]);
		  page.drawImage(img, {
			x: 0,
			y: 0,
			width: allPages[i].width,
			height: allPages[i].height,
		  });
		}
  
		const pdfBytes = await newPdf.save();
		const compressedFileName = currentFileName.replace('.pdf', '_compressed.pdf');
		download(new Blob([pdfBytes]), compressedFileName, "application/pdf");
  
	  } catch (error) {
		console.error("Error compressing PDF:", error);
		alert("Error compressing PDF: " + error.message);
	  }
  
	  progressDiv.style.display = 'none';
	  closeDialog('compress-dialog');
	}
  
	async function compressImageToSizeDataUri(dataUri, targetSizeInBytes, minQuality = 0.85) {
	  return new Promise(async (resolve, reject) => {
		const img = new Image();
		img.src = dataUri;
  
		img.onload = async function () {
		  const canvas = document.createElement('canvas');
		  const ctx = canvas.getContext('2d');
  
		  canvas.width = img.width;
		  canvas.height = img.height;
  
		  ctx.drawImage(img, 0, 0, img.width, img.height);
  
		  let compressionQuality = 0.95;
		  let compressedDataUrl = '';
		  let currentSize = Infinity;
  
		  while (currentSize > targetSizeInBytes && compressionQuality > minQuality) {
			compressedDataUrl = canvas.toDataURL('image/jpeg', compressionQuality);
			currentSize = compressedDataUrl.length;
  
			if (currentSize > targetSizeInBytes) {
			  compressionQuality -= 0.05;
			}
		  }
  
		  if (currentSize > targetSizeInBytes) {
			compressedDataUrl = canvas.toDataURL('image/jpeg', minQuality);
		  }
  
		  resolve(compressedDataUrl);
		};
  
		img.onerror = function (error) {
		  reject(error);
		};
	  });
	}
  
	let viewerElement;
	onMount(() => {
	  viewerElement = document.getElementById('viewer');
	  viewerElement.addEventListener('dragover', (e) => {
		e.preventDefault();
		e.stopPropagation();
		viewerElement.style.backgroundColor = '#e0e0e0';
	  });
  
	  viewerElement.addEventListener('dragleave', (e) => {
		e.preventDefault();
		e.stopPropagation();
		viewerElement.style.backgroundColor = '#ffffff';
	  });
  
	  viewerElement.addEventListener('drop', (e) => {
		e.preventDefault();
		e.stopPropagation();
		viewerElement.style.backgroundColor = '#ffffff';
  
		if (e.dataTransfer.files.length > 0) {
		  const file = e.dataTransfer.files[0];
		  if (file.type === 'application/pdf') {
			const reader = new FileReader();
			reader.onload = (e) => {
			  loadPDF(e.target.result, file.name);
			};
			reader.readAsArrayBuffer(file);
		  } else {
			alert('Please drop a PDF file.');
		  }
		}
	  });
	});
	let overlayElement; // To store a reference to the overlay element
	onMount(() => {
		// ... (existing onMount code for #viewer)

		overlayElement = document.getElementById('overlay');
		if (overlayElement) {
			overlayElement.addEventListener('dragover', (e) => {
				e.preventDefault();
				e.stopPropagation();
				overlayElement.style.backgroundColor = '#e0e0e0'; // Visual feedback on dragover
			});

			overlayElement.addEventListener('dragleave', (e) => {
				e.preventDefault();
				e.stopPropagation();
				overlayElement.style.backgroundColor = 'rgba(22, 56, 61, 0.9)'; // Reset background color
			});

			overlayElement.addEventListener('drop', (e) => {
				e.preventDefault();
				e.stopPropagation();
				overlayElement.style.backgroundColor = 'rgba(22, 56, 61, 0.9)'; // Reset background color

				if (e.dataTransfer.files.length > 0) {
					const file = e.dataTransfer.files[0];
					if (file.type === 'application/pdf') {
						const reader = new FileReader();
						reader.onload = (e) => {
							loadPDF(e.target.result, file.name);
						};
						reader.readAsArrayBuffer(file);
					} else {
						alert('Please drop a PDF file.');
					}
				}
			});
		}
	});
  </script>
  
  <main>
	{#if showOverlay}
	<div id="overlay">
		<div class="overlay-content">
			<header>
				<h1 class="privacy-title">PDF PRIVACY</h1>
				<p class="privacy-subtitle">
					<span class="highlight">DON'T</span> UPLOAD YOUR PDFS TO <span class="highlight">SHADY SERVERS</span>
				</p>
			</header>

			<div class="feature-list">
				<span class="highlight">COMPRESS</span> PDF, 
				<span class="highlight">MERGE</span> PDFS, 
				<span class="highlight">PDF TO IMAGES</span> CONVERSION, 
				<span class="highlight">SPLIT</span> PDF <br>
				USING THE <span class="highlight"> ON-BROWSER TECHNOLOGY</span> of PDFPRIVACY.COM
			</div>

			<button class="select-button" on:click={handleOpenNewClick}>
				SELECT PDF
			</button>
			<div class="drag-drop-area">
				<p class="drag-drop-text">Or Drag And Drop PDF</p>
			</div>
		</div>
	</div>
	{/if}

	<header>PDF Privacy : Don't upload your PDFs to shady Servers</header>
	<div id="toolbar">
	  <input type="file" id="fileInput" accept=".pdf,application/pdf" style="display: none;" on:change={handleFileInputChange} />
	  <button class="button" id="btn-open" on:click={handleOpenNewClick}>Open New</button>
	  <button class="button" id="btn-pdf2png" on:click={handlePdfToImageClick}>PDF to Image</button>
	  <button class="button" id="btn-split" on:click={handleSplitPdfClick}>Split PDF</button>
	  <button class="button" id="btn-merge" on:click={handleMergePdfClick}>Merge PDF</button>
	  <button class="button" id="btn-compress" on:click={handleCompressPdfClick}>Compress PDF</button>
	</div>
	<div id="viewer">
	  
	</div>
  
	<div id="pdf2png-dialog" class="dialog">
	  <h3>Convert to PNG</h3>
	  <label>
		<input type="radio" id="png-all-pages" name="png-pages" value="all" checked />
		All Pages
	  </label>
	  <label>
		<input type="radio" id="png-specific-pages" name="png-pages" value="specific" />
		Select Pages:
		<input type="text" id="png-page-input" placeholder="e.g., 1,3-5,7" />
	  </label>
	  <button id="png-convert-btn" on:click={handlePngConvertClick}>Convert</button>
	  <button on:click={() => closeDialog('pdf2png-dialog')}>Cancel</button>
	</div>
  
	<div id="split-dialog" class="dialog">
	  <h3>Split PDF</h3>
	  <label>
		Split at Page:
		<input type="number" id="split-page-input" min="2" placeholder="Enter page number" />
	  </label>
	  <button id="split-confirm-btn" on:click={handleSplitConfirmClick}>Split</button>
	  <button on:click={() => closeDialog('split-dialog')}>Cancel</button>
	</div>
  
	<div id="merge-dialog" class="dialog">
	  <h3>Merge PDFs</h3>
	  <div id="merge-files-list"></div>
	  <input type="file" id="merge-file-input" accept="application/pdf" style="display: none;" multiple on:change={handleMergeFileInputChange} />
	  <button id="add-merge-file-btn" on:click={handleAddMergeFileClick}>Upload Additional</button>
	  <button id="merge-btn" on:click={handleMergeButtonClick}>Merge</button>
	  <button on:click={() => closeDialog('merge-dialog')}>Cancel</button>
	</div>
  
	<div id="compress-dialog" class="dialog">
	  <h3>Compress PDF</h3>
	  <div>
		<label>
		  <input type="radio" name="compression-type" value="quality" checked /> Compress by Quality
		</label>
	  </div>
	  <div id="quality-options">
		<label>
		  <input type="radio" name="compression-quality" value="0.95" checked /> Low Compression
		</label>
		<label>
		  <input type="radio" name="compression-quality" value="0.9" /> Medium Compression
		</label>
		<label>
		  <input type="radio" name="compression-quality" value="0.85" /> High Compression
		</label>
	  </div>
	  <div id="size-options" style="display: none;">
		<label>
		  Target Size (KB): <input type="number" id="target-size" min="1" />
		</label>
	  </div>
	  <div id="compression-progress" style="display: none;">
		<progress id="compression-progress-bar" value="0" max="100"></progress>
		<span id="compression-progress-text">0%</span>
	  </div>
	  <button id="compress-confirm-btn" on:click={handleCompressConfirmClick}>Compress</button>
	  <button on:click={() => closeDialog('compress-dialog')}>Cancel</button>
	</div>
  </main>
  
  <style>
	:global(body) {
	  font-family: Arial, sans-serif;
	  background-color: #16383D;
	  color: white;
	  margin: 0;
	  padding: 0;
	}
  
	main {
	  display: flex;
	  flex-direction: column;
	  height: 100vh;
	}
  
	header {
	  background-color: #16383D;
	  color: white;
	  padding: 20px;
	  font-size: 24px;
	  text-align: center;
	}
  
	#toolbar {
	  background-color: #16383D;
	  padding: 10px;
	  display: flex;
	  justify-content: center;
	  gap: 10px;
	}
  
	.button {
	  padding: 10px 15px;
	  border: none;
	  background-color: white;
	  color: #16383D;
	  cursor: pointer;
	  border-radius: 5px;
	  font-size: 14px;
	  font-weight: bold;
	  transition: background-color 0.3s;
	}
  
	.button:hover {
	  background-color: #e0e0e0;
	}
  
	#viewer {
	  flex: 1;
	  background-color: white;
	  overflow: auto;
	  display: flex;
	  align-items: center;
	  justify-content: center;
	  color: #333;
	}
  
	.dialog {
	  display: none;
	  position: fixed;
	  left: 50%;
	  top: 50%;
	  transform: translate(-50%, -50%);
	  background-color: #16383D;
	  color: white;
	  border: none;
	  padding: 20px;
	  border-radius: 10px;
	  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
	  max-width: 90%;
	  width: 300px;
	}
  
	.dialog h3 {
	  margin-top: 0;
	  margin-bottom: 15px;
	  font-size: 18px;
	}
  
	.dialog label {
	  display: block;
	  margin-bottom: 10px;
	}
  
	.dialog input[type="radio"] {
	  margin-right: 5px;
	}
  
	.dialog input[type="text"],
	.dialog input[type="number"] {
	  width: 100%;
	  padding: 5px;
	  margin-top: 5px;
	  border: 1px solid rgba(255, 255, 255, 0.3);
	  background-color: rgba(255, 255, 255, 0.1);
	  color: white;
	  border-radius: 3px;
	}
  
	.dialog button {
	  background-color: white;
	  color: #16383D;
	  border: none;
	  padding: 8px 15px;
	  margin-right: 10px;
	  margin-top: 10px;
	  border-radius: 3px;
	  cursor: pointer;
	  font-weight: bold;
	  transition: background-color 0.3s;
	}
  
	.dialog button:hover {
	  background-color: #e0e0e0;
	}
  
	#merge-files-list {
	  margin-bottom: 15px;
	}
  
	.merge-file-item {
	  display: flex;
	  justify-content: space-between;
	  align-items: center;
	  margin-bottom: 5px;
	  background-color: rgba(255, 255, 255, 0.1);
	  padding: 5px 10px;
	  border-radius: 3px;
	}
  
	.remove-file {
	  background-color: #ff4d4d;
	  color: white;
	  border: none;
	  padding: 2px 5px;
	  cursor: pointer;
	  border-radius: 3px;
	}
  
	#compression-progress {
	  margin-top: 15px;
	}
  
	#compression-progress-bar {
	  width: 100%;
	  height: 20px;
	}
  
	#compression-progress-text {
	  display: block;
	  text-align: center;
	  margin-top: 5px;
	}
  
	@media (max-width: 600px) {
	  header {
		font-size: 20px;
		padding: 15px;
	  }
  
	  #toolbar {
		flex-wrap: wrap;
	  }
  
	  .button {
		width: calc(50% - 5px);
		margin-bottom: 10px;
	  }
	}
	#overlay {
	position: fixed;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	background-color: rgba(22, 56, 61, 0.9); /* Changed to white background */
	display: flex;
	align-items: center;
	justify-content: center;
	z-index: 1000;
}

.overlay-content {
	text-align: center;
	font-family: 'Inter', sans-serif;
	max-width: 1200px;
	width: 80%; /* Default: Desktop */
	padding: 2rem;
	border-radius: 8px;
	/* Removed backdrop */ 
}

@media (max-width: 768px) {
	.overlay-content {
		width: 90%; /* Tablet */
	}
}

@media (max-width: 1024px) {
	.overlay-content {
		width: 95%; /* Mobile */
	}
}

header {
	margin-bottom: 2rem;
	background-color: #16383D; /* Teal background for the header */
	color: white; /* White text color for the header */
	padding: 1rem; 
	border-radius: 8px; 
}

.privacy-title {
	font-size: 48px; /* Default: Desktop */
	text-transform: uppercase;
	font-weight: 700;
	margin-bottom: 1rem;
	color: #ffffff;
}

@media (max-width: 768px) {
	.privacy-title {
		font-size: 36px; /* Tablet */
	}
}

@media (max-width: 1024px) {
	.privacy-title {
		font-size: 28px; /* Mobile */
	}
}

.privacy-subtitle {
	font-size: 24px; /* Default: Desktop */
	font-weight: 500;
	margin-bottom: 2rem;
	color: #ffffff;
}

@media (max-width: 768px) {
	.privacy-subtitle {
		font-size: 20px; /* Tablet */
	}
}

@media (max-width: 1024px) {
	.privacy-subtitle {
		font-size: 18px; /* Mobile */
	}
}

.feature-list {
	/* Removed grid styles */
	text-align: center; /* Center the text */
	margin-bottom: 2rem;
	color: white; /* Teal color for feature items */
}

.feature-item {
	display: block; /* Changed to block for better text flow */
	font-size: 18px;
	color: #16383D; /* Teal color for feature items */
	margin-bottom: 0.5rem; /* Add space between feature items */
}

.highlight {
	color: #ffd700;
}

.select-button {
	background-color: #16383D; /* Teal color for the button */ 
	color: #ffffff;
	border: none;
	padding: 1rem 2rem;
	border-radius: 8px;
	font-size: 20px;
	font-weight: 600;
	cursor: pointer;
	transition: background-color 0.3s ease;
	box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
	animation: pulse 1.5s infinite;
}

.select-button:hover {
	filter: brightness(110%);
}

.drag-drop-area {
	border: 2px dashed #ffffff;
	border-radius: 8px;
	padding: 2rem;
	text-align: center;
	margin-top: 2rem;
}

.drag-drop-text {
	font-size: 16px;
	color: #ffffff;
}

/* Pulse Animation */
@keyframes pulse {
	0% {
		transform: scale(1);
	}
	50% {
		transform: scale(1.05);
	}
	100% {
		transform: scale(1);
	}
}

  </style>
  
  <svelte:head>
	<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.9.359/pdf.min.js"></script>
	<script src="https://unpkg.com/pdf-lib@1.17.1"></script>
	<script src="https://unpkg.com/downloadjs@1.4.7"></script>
  </svelte:head>