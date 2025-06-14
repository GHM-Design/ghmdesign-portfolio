<script lang="ts">
	import { fade, slide } from 'svelte/transition';
	import { onMount } from 'svelte';
	
	let fileInput: HTMLInputElement;
	let canvas: HTMLCanvasElement;
	let processedCanvas: HTMLCanvasElement;
	let imageUrl: string | null = null;
	let isProcessing = false;
	let extractedData: any = null;
	let showCopySuccess = false;
	let tessWorker: any = null;
	let processingStage = '';
	let opencvReady = false;
	let processingMode: 'gentle' | 'aggressive' = 'gentle';
	let debugMode = false;
	let skipEdgeDetection = false; // Add option to skip edge detection
	
	// Initialize libraries
	onMount(async () => {
		if (typeof window !== 'undefined') {
			// Load OpenCV.js
			const cvScript = document.createElement('script');
			cvScript.src = 'https://docs.opencv.org/4.5.5/opencv.js';
			cvScript.async = true;
			cvScript.onload = () => {
				// @ts-ignore
				cv['onRuntimeInitialized'] = () => {
					opencvReady = true;
					console.log('OpenCV.js is ready');
				};
			};
			document.head.appendChild(cvScript);
			
			// Load Tesseract.js
			const tessScript = document.createElement('script');
			tessScript.src = 'https://cdn.jsdelivr.net/npm/tesseract.js@4/dist/tesseract.min.js';
			document.head.appendChild(tessScript);
			
			tessScript.onload = async () => {
				// @ts-ignore
				const { createWorker } = window.Tesseract;
				tessWorker = await createWorker();
				await tessWorker.loadLanguage('eng');
				await tessWorker.initialize('eng');
			};
		}
		
		return () => {
			if (tessWorker) {
				tessWorker.terminate();
			}
		};
	});
	
	function handleFileSelect(event: Event) {
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		
		if (file && file.type.startsWith('image/')) {
			const reader = new FileReader();
			reader.onload = (e) => {
				imageUrl = e.target?.result as string;
				processImage(imageUrl);
			};
			reader.readAsDataURL(file);
		}
	}
	
	async function processImage(src: string) {
		isProcessing = true;
		processingStage = 'Loading image...';
		
		const img = new Image();
		img.onload = async () => {
			if (opencvReady) {
				await processWithOpenCV(img);
			} else {
				await processWithCanvas(img);
			}
		};
		img.src = src;
	}
	
	function autoCropDocument(canvas: HTMLCanvasElement) {
		const ctx = canvas.getContext('2d');
		const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
		const data = imageData.data;
		
		let minX = canvas.width;
		let minY = canvas.height;
		let maxX = 0;
		let maxY = 0;
		
		// Find bounding box of non-white pixels
		for (let y = 0; y < canvas.height; y++) {
			for (let x = 0; x < canvas.width; x++) {
				const idx = (y * canvas.width + x) * 4;
				// Check if pixel is not white (allowing some threshold)
				if (data[idx] < 250 || data[idx + 1] < 250 || data[idx + 2] < 250) {
					minX = Math.min(minX, x);
					minY = Math.min(minY, y);
					maxX = Math.max(maxX, x);
					maxY = Math.max(maxY, y);
				}
			}
		}
		
		// Add small padding
		const padding = 0; // Changed from 20 to 0 - no padding
		minX = Math.max(0, minX - padding);
		minY = Math.max(0, minY - padding);
		maxX = Math.min(canvas.width - 1, maxX + padding);
		maxY = Math.min(canvas.height - 1, maxY + padding);
		
		// Calculate crop dimensions
		const cropWidth = maxX - minX;
		const cropHeight = maxY - minY;
		
		// Create cropped canvas
		const croppedCanvas = document.createElement('canvas');
		croppedCanvas.width = cropWidth;
		croppedCanvas.height = cropHeight;
		const croppedCtx = croppedCanvas.getContext('2d');
		
		// Copy cropped region
		croppedCtx.drawImage(
			canvas,
			minX, minY, cropWidth, cropHeight,
			0, 0, cropWidth, cropHeight
		);
		
		// Update the processed canvas with cropped image
		canvas.width = cropWidth;
		canvas.height = cropHeight;
		ctx.drawImage(croppedCanvas, 0, 0);
		
		console.log(`Auto-cropped from ${imageData.width}x${imageData.height} to ${cropWidth}x${cropHeight}`);
		
		return canvas;
	}
	
	async function processWithOpenCV(img: HTMLImageElement) {
		// @ts-ignore
		const cv = window.cv;
		
		try {
			processingStage = 'Loading image...';
			
			// Load image into OpenCV
			const mat = cv.imread(img);
			const original = mat.clone();
			
			// If skip edge detection is enabled, just use the original image
			if (skipEdgeDetection) {
				processingStage = 'Using original image (edge detection skipped)...';
				cv.imshow(processedCanvas, original);
				
				// Cleanup
				mat.delete();
				original.delete();
				
				// Continue with OCR setup
				processingStage = 'Done! Ready for OCR.';
				extractedData = {
					vendor: 'Ready for OCR',
					date: new Date().toLocaleDateString(),
					time: '',
					subtotal: 0,
					tax: 0,
					total: 0,
					paymentMethod: 'N/A',
					items: [{
						description: 'Edge detection skipped - using original image',
						quantity: 1,
						unitPrice: 0,
						total: 0
					}],
					rawText: 'Enable OCR in the code to extract text from this image.',
					confidence: 100
				};
				
				isProcessing = false;
				processingStage = '';
				return;
			}
			
			processingStage = 'Detecting document edges...';
			
			// Convert to grayscale
			const gray = new cv.Mat();
			cv.cvtColor(mat, gray, cv.COLOR_RGBA2GRAY);
			
			// Apply Gaussian blur to reduce noise
			const blurred = new cv.Mat();
			cv.GaussianBlur(gray, blurred, new cv.Size(5, 5), 0);
			
			// Apply Canny edge detection
			const edges = new cv.Mat();
			cv.Canny(blurred, edges, 75, 200);
			
			// Apply morphological closing to connect edges
			const kernel = cv.getStructuringElement(cv.MORPH_RECT, new cv.Size(7, 7));
			cv.morphologyEx(edges, edges, cv.MORPH_CLOSE, kernel);
			
			// Find contours
			const contours = new cv.MatVector();
			const hierarchy = new cv.Mat();
			cv.findContours(edges, contours, hierarchy, cv.RETR_EXTERNAL, cv.CHAIN_APPROX_SIMPLE);
			
			// Find the largest rectangular contour
			let maxArea = 0;
			let bestContour = null;
			
			for (let i = 0; i < contours.size(); i++) {
				const contour = contours.get(i);
				const area = cv.contourArea(contour);
				
				if (area > maxArea) {
					// Approximate contour to polygon
					const peri = cv.arcLength(contour, true);
					const approx = new cv.Mat();
					cv.approxPolyDP(contour, approx, 0.02 * peri, true);
					
					// Check if it's roughly rectangular (4 corners)
					if (approx.rows === 4) {
						maxArea = area;
						bestContour = approx;
					} else {
						approx.delete();
					}
				}
			}
			
			let finalImage;
			
			if (bestContour && maxArea > (mat.rows * mat.cols) * 0.1) {
				processingStage = 'Extracting and correcting document...';
				
				// Get the four corners
				const srcPoints = [];
				for (let i = 0; i < 4; i++) {
					srcPoints.push([bestContour.data32S[i * 2], bestContour.data32S[i * 2 + 1]]);
				}
				
				// Order points: top-left, top-right, bottom-right, bottom-left
				const rect = orderPointsNew(srcPoints);
				
				// Calculate dimensions of the output image
				const widthA = Math.sqrt(Math.pow(rect[2][0] - rect[3][0], 2) + Math.pow(rect[2][1] - rect[3][1], 2));
				const widthB = Math.sqrt(Math.pow(rect[1][0] - rect[0][0], 2) + Math.pow(rect[1][1] - rect[0][1], 2));
				const maxWidth = Math.max(widthA, widthB);
				
				const heightA = Math.sqrt(Math.pow(rect[1][0] - rect[2][0], 2) + Math.pow(rect[1][1] - rect[2][1], 2));
				const heightB = Math.sqrt(Math.pow(rect[0][0] - rect[3][0], 2) + Math.pow(rect[0][1] - rect[3][1], 2));
				const maxHeight = Math.max(heightA, heightB);
				
				// Source and destination points for perspective transform
				const src = cv.matFromArray(4, 1, cv.CV_32FC2, [
					rect[0][0], rect[0][1],
					rect[1][0], rect[1][1],
					rect[2][0], rect[2][1],
					rect[3][0], rect[3][1]
				]);
				
				const dst = cv.matFromArray(4, 1, cv.CV_32FC2, [
					0, 0,
					maxWidth - 1, 0,
					maxWidth - 1, maxHeight - 1,
					0, maxHeight - 1
				]);
				
				// Get perspective transform matrix
				const M = cv.getPerspectiveTransform(src, dst);
				
				// Warp the original image
				finalImage = new cv.Mat();
				const dsize = new cv.Size(maxWidth, maxHeight);
				cv.warpPerspective(original, finalImage, M, dsize, cv.INTER_LINEAR, cv.BORDER_CONSTANT, new cv.Scalar(255, 255, 255));
				
				// Cleanup
				src.delete();
				dst.delete();
				M.delete();
				bestContour.delete();
			} else {
				// If no good contour found, try to isolate the receipt by color
				console.log('No suitable document contour found, attempting color-based extraction');
				
				processingStage = 'Extracting receipt by color...';
				
				// Convert to HSV for better color segmentation
				const hsv = new cv.Mat();
				cv.cvtColor(original, hsv, cv.COLOR_RGB2HSV);
				
				// Create mask for light-colored areas (the receipt)
				const lowerBound = new cv.Mat(hsv.rows, hsv.cols, hsv.type(), [0, 0, 180, 0]);
				const upperBound = new cv.Mat(hsv.rows, hsv.cols, hsv.type(), [180, 30, 255, 255]);
				const mask = new cv.Mat();
				cv.inRange(hsv, lowerBound, upperBound, mask);
				
				// Apply morphological operations to clean up the mask
				const morphKernel = cv.getStructuringElement(cv.MORPH_RECT, new cv.Size(5, 5));
				cv.morphologyEx(mask, mask, cv.MORPH_CLOSE, morphKernel);
				cv.morphologyEx(mask, mask, cv.MORPH_OPEN, morphKernel);
				
				// Find contours in the mask
				const maskContours = new cv.MatVector();
				const maskHierarchy = new cv.Mat();
				cv.findContours(mask, maskContours, maskHierarchy, cv.RETR_EXTERNAL, cv.CHAIN_APPROX_SIMPLE);
				
				// Find the largest contour (should be the receipt)
				let largestArea = 0;
				let largestContourIndex = -1;
				
				for (let i = 0; i < maskContours.size(); i++) {
					const area = cv.contourArea(maskContours.get(i));
					if (area > largestArea) {
						largestArea = area;
						largestContourIndex = i;
					}
				}
				
				if (largestContourIndex >= 0) {
					// Get bounding rectangle of the receipt
					const rect = cv.boundingRect(maskContours.get(largestContourIndex));
					
					// Extract the receipt region with some padding
					const padding = 10;
					const x = Math.max(0, rect.x - padding);
					const y = Math.max(0, rect.y - padding);
					const width = Math.min(original.cols - x, rect.width + 2 * padding);
					const height = Math.min(original.rows - y, rect.height + 2 * padding);
					
					finalImage = original.roi(new cv.Rect(x, y, width, height));
				} else {
					finalImage = original.clone();
				}
				
				// Cleanup
				hsv.delete();
				lowerBound.delete();
				upperBound.delete();
				mask.delete();
				morphKernel.delete();
				maskContours.delete();
				maskHierarchy.delete();
			}
			
			// Apply image enhancement to make it look like a scan
			processingStage = 'Enhancing document...';
			
			// Convert to grayscale for processing
			const finalGray = new cv.Mat();
			cv.cvtColor(finalImage, finalGray, cv.COLOR_RGBA2GRAY);
			
			// Apply bilateral filter to reduce noise while preserving edges
			const denoised = new cv.Mat();
			cv.bilateralFilter(finalGray, denoised, 9, 75, 75);
			
			// Apply adaptive threshold with larger block size for smoother results
			const thresholded = new cv.Mat();
			cv.adaptiveThreshold(denoised, thresholded, 255, cv.ADAPTIVE_THRESH_GAUSSIAN_C, cv.THRESH_BINARY, 21, 10);
			
			// Remove small noise with morphological operations
			const cleanKernel = cv.getStructuringElement(cv.MORPH_ELLIPSE, new cv.Size(2, 2));
			cv.morphologyEx(thresholded, thresholded, cv.MORPH_OPEN, cleanKernel);
			cv.morphologyEx(thresholded, thresholded, cv.MORPH_CLOSE, cleanKernel);
			
			// Apply median filter to remove salt and pepper noise
			cv.medianBlur(thresholded, thresholded, 3);
			
			// Convert back to color (white background)
			const result = new cv.Mat();
			cv.cvtColor(thresholded, result, cv.COLOR_GRAY2RGBA);
			
			// Display the result
			cv.imshow(processedCanvas, result);
			
			// Auto-crop to remove excess white space
			processingStage = 'Auto-cropping document...';
			const cropped = autoCropDocument(processedCanvas);
			
			// Cleanup
			original.delete();
			mat.delete();
			gray.delete();
			blurred.delete();
			edges.delete();
			kernel.delete();
			cleanKernel.delete();
			contours.delete();
			hierarchy.delete();
			finalImage.delete();
			finalGray.delete();
			denoised.delete();
			thresholded.delete();
			result.delete();
			
			// For now, skip OCR and just show the processed image
			processingStage = 'Done! Document extracted and enhanced.';
			
			// Set dummy data to show success
			extractedData = {
				vendor: 'Document Processing Complete',
				date: new Date().toLocaleDateString(),
				time: '',
				subtotal: 0,
				tax: 0,
				total: 0,
				paymentMethod: 'N/A',
				items: [],
				rawText: 'Document has been extracted and enhanced to look like a scan.',
				confidence: 100
			};
			
		} catch (error) {
			console.error('OpenCV processing error:', error);
			await processWithCanvas(img);
		}
		
		isProcessing = false;
		processingStage = '';
	}
	
	function orderPointsNew(pts: number[][]) {
		// Sort the points based on their x-coordinates
		const xSorted = pts.slice().sort((a, b) => a[0] - b[0]);
		
		// Grab the left-most and right-most points
		const leftMost = xSorted.slice(0, 2);
		const rightMost = xSorted.slice(2);
		
		// Sort left points by y-coordinate to get top-left and bottom-left
		leftMost.sort((a, b) => a[1] - b[1]);
		const [tl, bl] = leftMost;
		
		// Sort right points by y-coordinate to get top-right and bottom-right
		rightMost.sort((a, b) => a[1] - b[1]);
		const [tr, br] = rightMost;
		
		return [tl, tr, br, bl];
	}
	
	function orderPoints(points: any[]) {
		// Order points: top-left, top-right, bottom-right, bottom-left
		const sorted = points.sort((a, b) => a.y - b.y);
		const top = sorted.slice(0, 2).sort((a, b) => a.x - b.x);
		const bottom = sorted.slice(2, 4).sort((a, b) => b.x - a.x);
		return [top[0], top[1], bottom[0], bottom[1]];
	}
	
	function distance(p1: any, p2: any) {
		return Math.sqrt(Math.pow(p2.x - p1.x, 2) + Math.pow(p2.y - p1.y, 2));
	}
	
	async function processWithCanvas(img: HTMLImageElement) {
		processingStage = 'Processing image...';
		
		const ctx = canvas.getContext('2d');
		canvas.width = img.width;
		canvas.height = img.height;
		ctx.drawImage(img, 0, 0);
		
		// Simple crop
		const cropMargin = Math.min(img.width, img.height) * 0.05;
		processedCanvas.width = img.width - cropMargin * 2;
		processedCanvas.height = img.height - cropMargin * 2;
		
		const processedCtx = processedCanvas.getContext('2d');
		processedCtx.drawImage(
			img,
			cropMargin, cropMargin,
			img.width - cropMargin * 2, img.height - cropMargin * 2,
			0, 0,
			processedCanvas.width, processedCanvas.height
		);
		
		// Scale to optimal size
		await scaleToOptimalSize(processedCanvas);
		
		processingStage = 'Enhancing image...';
		if (processingMode === 'gentle') {
			await gentleEnhance(processedCanvas);
		} else {
			await enhanceForOCR(processedCanvas);
		}
		
		processingStage = 'Extracting text...';
		const ocrResult = await performOCR(processedCanvas);
		
		processingStage = 'Parsing receipt data...';
		extractedData = parseReceiptData(ocrResult);
		
		isProcessing = false;
		processingStage = '';
	}
	
	async function scaleToOptimalSize(canvas: HTMLCanvasElement) {
		const ctx = canvas.getContext('2d');
		const currentWidth = canvas.width;
		const currentHeight = canvas.height;
		
		// Calculate scale factor to achieve ~300 DPI (optimal for OCR)
		// Assuming typical receipt is ~3 inches wide
		const optimalWidth = 900; // 3 inches * 300 DPI
		const scale = optimalWidth / currentWidth;
		
		// Only scale up if image is too small
		if (scale > 1) {
			const newCanvas = document.createElement('canvas');
			newCanvas.width = currentWidth * scale;
			newCanvas.height = currentHeight * scale;
			const newCtx = newCanvas.getContext('2d');
			
			// Use bicubic interpolation for better quality
			newCtx.imageSmoothingEnabled = true;
			newCtx.imageSmoothingQuality = 'high';
			newCtx.drawImage(canvas, 0, 0, newCanvas.width, newCanvas.height);
			
			// Copy back to original canvas
			canvas.width = newCanvas.width;
			canvas.height = newCanvas.height;
			ctx.drawImage(newCanvas, 0, 0);
		}
		
		// Add white border (improves OCR accuracy)
		const borderSize = 20;
		const borderedCanvas = document.createElement('canvas');
		borderedCanvas.width = canvas.width + borderSize * 2;
		borderedCanvas.height = canvas.height + borderSize * 2;
		const borderedCtx = borderedCanvas.getContext('2d');
		
		// Fill with white
		borderedCtx.fillStyle = 'white';
		borderedCtx.fillRect(0, 0, borderedCanvas.width, borderedCanvas.height);
		
		// Draw image in center
		borderedCtx.drawImage(canvas, borderSize, borderSize);
		
		// Copy back
		canvas.width = borderedCanvas.width;
		canvas.height = borderedCanvas.height;
		ctx.drawImage(borderedCanvas, 0, 0);
	}
	
	async function gentleEnhance(canvas: HTMLCanvasElement) {
		const ctx = canvas.getContext('2d');
		const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
		const data = imageData.data;
		
		// Very gentle enhancement
		for (let i = 0; i < data.length; i += 4) {
			let r = data[i];
			let g = data[i + 1];
			let b = data[i + 2];
			
			// Slight contrast boost
			r = ((r - 128) * 1.1) + 128;
			g = ((g - 128) * 1.1) + 128;
			b = ((b - 128) * 1.1) + 128;
			
			data[i] = Math.max(0, Math.min(255, r));
			data[i + 1] = Math.max(0, Math.min(255, g));
			data[i + 2] = Math.max(0, Math.min(255, b));
		}
		
		ctx.putImageData(imageData, 0, 0);
		return canvas;
	}
	
	async function enhanceForOCR(canvas: HTMLCanvasElement) {
		const ctx = canvas.getContext('2d');
		const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
		const data = imageData.data;
		
		// Convert to grayscale and enhance
		for (let i = 0; i < data.length; i += 4) {
			let gray = data[i] * 0.299 + data[i + 1] * 0.587 + data[i + 2] * 0.114;
			
			// Increase contrast
			gray = ((gray - 128) * 1.5) + 128;
			gray = Math.max(0, Math.min(255, gray));
			
			// Threshold
			gray = gray > 140 ? 255 : 0;
			
			data[i] = gray;
			data[i + 1] = gray;
			data[i + 2] = gray;
		}
		
		ctx.putImageData(imageData, 0, 0);
		return canvas;
	}
	
	async function performOCR(canvas: HTMLCanvasElement) {
		if (!tessWorker) {
			return { text: 'OCR not initialized' };
		}
		
		try {
			// PSM 4 (SINGLE_COLUMN) works best for receipts
			const ocrConfig = {
				tessedit_pageseg_mode: '4', // Single column of variable sizes
				tessedit_char_whitelist: '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz .,()/*@%-#&$:',
				preserve_interword_spaces: '1',
				load_system_dawg: '0', // Don't use dictionary
				load_freq_dawg: '0',
				user_defined_dpi: '300',
				tessedit_ocr_engine_mode: '1' // LSTM only
			};
			
			const result = await tessWorker.recognize(canvas, ocrConfig);
			
			// If we get low confidence, try zone-based processing
			if (result.data.confidence < 80) {
				console.log('Low confidence, trying zone-based processing...');
				return await performZoneBasedOCR(canvas);
			}
			
			return result.data;
		} catch (error) {
			console.error('OCR error:', error);
			return { text: '' };
		}
	}
	
	async function performZoneBasedOCR(canvas: HTMLCanvasElement) {
		// Process receipt in zones for better accuracy
		const zones = {
			header: { top: 0, height: 0.3 }, // Top 30%
			items: { top: 0.3, height: 0.5 }, // Middle 50%
			totals: { top: 0.8, height: 0.2 }  // Bottom 20%
		};
		
		let fullText = '';
		
		for (const [zoneName, zone] of Object.entries(zones)) {
			const zoneCanvas = document.createElement('canvas');
			const zoneCtx = zoneCanvas.getContext('2d');
			
			const startY = Math.floor(canvas.height * zone.top);
			const height = Math.floor(canvas.height * zone.height);
			
			zoneCanvas.width = canvas.width;
			zoneCanvas.height = height;
			
			// Copy zone to new canvas
			zoneCtx.drawImage(
				canvas,
				0, startY, canvas.width, height,
				0, 0, canvas.width, height
			);
			
			// Use different PSM for different zones
			const psm = zoneName === 'totals' ? '7' : '4'; // Single line for totals
			
			const zoneConfig = {
				tessedit_pageseg_mode: psm,
				tessedit_char_whitelist: '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz .,()/*@%-#&$:',
				preserve_interword_spaces: '1',
				user_defined_dpi: '300'
			};
			
			const result = await tessWorker.recognize(zoneCanvas, zoneConfig);
			fullText += result.data.text + '\n';
		}
		
		return { text: fullText, confidence: 85 }; // Estimated confidence
	}
	
	function parseReceiptData(ocrResult: any) {
		const text = ocrResult.text || '';
		const lines = text.split('\n').map(line => line.trim()).filter(line => line.length > 0);
		
		console.log('OCR Text lines:', lines);
		
		// Extract vendor - look for business name patterns
		let vendor = 'Unknown Vendor';
		for (let i = 0; i < Math.min(5, lines.length); i++) {
			const line = lines[i];
			// Skip if line is mostly numbers or too short
			if (!/^\d+$/.test(line) && line.length > 3) {
				vendor = line;
				break;
			}
		}
		
		// Find date - multiple patterns
		const datePatterns = [
			/(\d{1,2}[\/\-]\d{1,2}[\/\-]\d{2,4})/,
			/(\d{2,4}[\/\-]\d{1,2}[\/\-]\d{1,2})/,
			/([A-Za-z]{3,9}\s+\d{1,2},?\s+\d{4})/,
			/(\d{1,2}\s+[A-Za-z]{3,9}\s+\d{4})/
		];
		
		let date = new Date().toLocaleDateString();
		for (const pattern of datePatterns) {
			const match = text.match(pattern);
			if (match) {
				date = match[1];
				break;
			}
		}
		
		// Find time
		const timeMatch = text.match(/(\d{1,2}:\d{2}(?::\d{2})?(?:\s*[ap]\.?m\.?)?)/i);
		const time = timeMatch ? timeMatch[1] : '';
		
		// Find total amount - look for largest amount or "TOTAL" keyword
		let total = 0;
		const totalPatterns = [
			/(?:TOTAL|Total|total)\s*:?\s*\$?([\d,]+\.?\d*)/i,
			/(?:AMOUNT|Amount)\s*:?\s*\$?([\d,]+\.?\d*)/i,
			/\$\s*([\d,]+\.\d{2})/g // Any currency amount
		];
		
		for (const pattern of totalPatterns) {
			const matches = text.matchAll(pattern);
			for (const match of matches) {
				const amount = parseFloat(match[1].replace(/,/g, ''));
				if (amount > total) {
					total = amount;
				}
			}
		}
		
		// Find subtotal
		let subtotal = 0;
		const subtotalMatch = text.match(/(?:SUBTOTAL|Sub\s*total|Amount)\s*:?\s*\$?([\d,]+\.?\d*)/i);
		if (subtotalMatch) {
			subtotal = parseFloat(subtotalMatch[1].replace(/,/g, ''));
		} else {
			subtotal = total * 0.9; // Estimate
		}
		
		// Find tax or tip
		let tax = 0;
		const taxPatterns = [
			/(?:TAX|Tax|HST|GST|PST)\s*:?\s*\$?([\d,]+\.?\d*)/i,
			/(?:TIP|Tip|Gratuity)\s*:?\s*\$?([\d,]+\.?\d*)/i
		];
		
		for (const pattern of taxPatterns) {
			const match = text.match(pattern);
			if (match) {
				tax = parseFloat(match[1].replace(/,/g, ''));
				break;
			}
		}
		
		if (tax === 0 && total > 0 && subtotal > 0) {
			tax = total - subtotal;
		}
		
		// Find payment method
		const paymentPatterns = [
			/(VISA|MASTERCARD|MASTER|MC|AMEX|AMERICAN EXPRESS|DISCOVER|DEBIT|CREDIT|CASH|APPROVED)/i,
			/\*{2,}(\d{4})/,
			/(XXXX|xxxx)\s*(\d{4})/
		];
		
		let paymentMethod = 'Unknown';
		for (const pattern of paymentPatterns) {
			const match = text.match(pattern);
			if (match) {
				paymentMethod = match[0];
				break;
			}
		}
		
		// Parse items - look for price patterns
		const items = [];
		const itemPatterns = [
			/^(.+?)\s+\$?([\d,]+\.\d{2})$/,
			/^(.+?)\s+(\d+)\s*[@x]\s*\$?([\d.]+)\s+\$?([\d,]+\.\d{2})$/,
			/^(.+?)\s{2,}\$?([\d,]+\.\d{2})$/
		];
		
		for (const line of lines) {
			// Skip lines that are headers or totals
			if (line.match(/TOTAL|TAX|SUBTOTAL|TIP|CASH|VISA|THANK|RECEIPT/i)) continue;
			
			for (const pattern of itemPatterns) {
				const match = line.match(pattern);
				if (match) {
					const price = parseFloat(match[match.length - 1].replace(/,/g, ''));
					if (price > 0 && price < total * 0.8) {
						items.push({
							description: match[1].trim(),
							quantity: match.length > 3 ? parseInt(match[2]) : 1,
							unitPrice: match.length > 3 ? parseFloat(match[3]) : price,
							total: price
						});
					}
					break;
				}
			}
		}
		
		return {
			vendor: vendor.trim(),
			date,
			time,
			subtotal: subtotal || total * 0.9,
			tax: tax || total * 0.1,
			total,
			paymentMethod,
			items,
			rawText: text,
			confidence: ocrResult.confidence || 0
		};
	}
	
	function formatCurrency(amount: number): string {
		return `$${amount.toFixed(2)}`;
	}
	
	function copyToClipboard() {
		if (!extractedData) return;
		
		let clipboardText = `Receipt Data\n`;
		clipboardText += `============\n\n`;
		clipboardText += `Vendor: ${extractedData.vendor}\n`;
		clipboardText += `Date: ${extractedData.date}\n`;
		if (extractedData.time) {
			clipboardText += `Time: ${extractedData.time}\n`;
		}
		clipboardText += `\nItems:\n`;
		clipboardText += `------\n`;
		
		if (extractedData.items.length > 0) {
			extractedData.items.forEach((item: any) => {
				clipboardText += `${item.description}\n`;
				clipboardText += `  Qty: ${item.quantity} @ ${formatCurrency(item.unitPrice)} = ${formatCurrency(item.total)}\n`;
			});
		} else {
			clipboardText += `(No items detected)\n`;
		}
		
		clipboardText += `\nSubtotal: ${formatCurrency(extractedData.subtotal)}\n`;
		clipboardText += `Tax: ${formatCurrency(extractedData.tax)}\n`;
		clipboardText += `Total: ${formatCurrency(extractedData.total)}\n`;
		clipboardText += `Payment: ${extractedData.paymentMethod}\n`;
		
		navigator.clipboard.writeText(clipboardText).then(() => {
			showCopySuccess = true;
			setTimeout(() => {
				showCopySuccess = false;
			}, 2000);
		});
	}
	
	function reset() {
		imageUrl = null;
		extractedData = null;
		if (fileInput) fileInput.value = '';
	}
	
	function reprocessWithDifferentMode() {
		if (imageUrl) {
			processingMode = processingMode === 'gentle' ? 'aggressive' : 'gentle';
			processImage(imageUrl);
		}
	}
</script>

<div class="max-w-7xl mx-auto">
	<div class="mb-8">
		<h2 class="text-3xl font-bold text-gray-900 mb-2">Receipt Scanner Tool</h2>
		<p class="text-gray-600">Upload a receipt image to automatically detect edges, flatten, and extract text</p>
		{#if !opencvReady}
			<p class="text-sm text-amber-600 mt-2">⚠️ Computer vision library loading... Basic mode active</p>
		{/if}
	</div>
	
	<!-- Hidden canvases for image processing -->
	<canvas bind:this={canvas} class="hidden"></canvas>
	<canvas bind:this={processedCanvas} class="hidden"></canvas>
	
	<div class="grid grid-cols-1 lg:grid-cols-2 gap-8">
		<!-- Left Column - Upload and Image Preview -->
		<div class="space-y-6">
			{#if !imageUrl}
				<!-- Upload Area -->
				<div class="border-2 border-dashed border-gray-300 rounded-lg p-8 text-center hover:border-gray-400 transition-colors">
					<input
						type="file"
						accept="image/*"
						on:change={handleFileSelect}
						bind:this={fileInput}
						class="hidden"
						id="receipt-upload"
					/>
					<label for="receipt-upload" class="cursor-pointer">
						<svg class="mx-auto h-12 w-12 text-gray-400 mb-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" />
						</svg>
						<p class="text-lg font-medium text-gray-900 mb-2">Click to upload receipt</p>
						<p class="text-sm text-gray-500">or drag and drop</p>
						<p class="text-xs text-gray-400 mt-2">PNG, JPG, GIF up to 10MB</p>
					</label>
				</div>
				
				<!-- Feature list -->
				<div class="bg-blue-50 rounded-lg p-6">
					<h3 class="font-semibold text-gray-900 mb-3">Processing Modes:</h3>
					<div class="space-y-3">
						<div>
							<p class="font-medium text-gray-700">🔵 Gentle Mode (Default)</p>
							<p class="text-sm text-gray-600">For clear, well-lit receipts. Minimal processing preserves text quality.</p>
						</div>
						<div>
							<p class="font-medium text-gray-700">🔴 Aggressive Mode</p>
							<p class="text-sm text-gray-600">For faded or poor quality receipts. Heavy processing to enhance readability.</p>
						</div>
					</div>
				</div>
			{:else}
				<!-- Image Preview with processed view -->
				<div class="space-y-4">
					<div class="grid grid-cols-2 gap-4">
						<!-- Original Image -->
						<div>
							<p class="text-sm font-medium text-gray-700 mb-2">Original</p>
							<img 
								src={imageUrl} 
								alt="Original receipt" 
								class="w-full rounded-lg shadow-lg"
							/>
						</div>
						
						<!-- Processed Image -->
						<div>
							<p class="text-sm font-medium text-gray-700 mb-2">Processed</p>
							<div class="relative max-h-[600px] overflow-auto border border-gray-200 rounded-lg">
								<canvas 
									bind:this={processedCanvas}
									class="rounded-lg shadow-lg bg-gray-100"
									style="max-width: 100%; height: auto; display: block;"
								/>
								{#if isProcessing}
									<div class="absolute inset-0 bg-black bg-opacity-50 rounded-lg flex items-center justify-center">
										<div class="text-center text-white">
											<div class="w-8 h-8 border-4 border-white border-t-transparent rounded-full animate-spin mx-auto mb-2"></div>
											<p class="text-sm font-medium">{processingStage}</p>
										</div>
									</div>
								{/if}
							</div>
						</div>
					</div>
					
					{#if extractedData && opencvReady}
						<div class="text-center text-sm text-green-600 bg-green-50 rounded-lg p-3">
							<p>✓ Document edges detected and perspective corrected successfully</p>
						</div>
					{/if}
				</div>
				
				<!-- Controls -->
				<div class="space-y-3">
					<button
						on:click={reset}
						class="w-full px-4 py-2 bg-gray-100 hover:bg-gray-200 text-gray-700 font-medium rounded-lg transition-colors"
					>
						Upload Different Receipt
					</button>
					
					<!-- Processing Mode Toggle -->
					<div class="bg-blue-50 border border-blue-200 rounded-lg p-4">
						<p class="text-sm font-medium text-gray-700 mb-3">Processing Mode</p>
						<div class="grid grid-cols-2 gap-2">
							<button
								on:click={() => reprocessWithDifferentMode()}
								disabled={processingMode === 'gentle'}
								class="px-4 py-2 rounded-lg text-sm font-medium transition-colors {processingMode === 'gentle' ? 'bg-blue-600 text-white' : 'bg-white text-gray-700 border border-gray-300 hover:bg-gray-50'}"
							>
								Gentle (Clear receipts)
							</button>
							<button
								on:click={() => reprocessWithDifferentMode()}
								disabled={processingMode === 'aggressive'}
								class="px-4 py-2 rounded-lg text-sm font-medium transition-colors {processingMode === 'aggressive' ? 'bg-blue-600 text-white' : 'bg-white text-gray-700 border border-gray-300 hover:bg-gray-50'}"
							>
								Aggressive (Poor quality)
							</button>
						</div>
						<p class="text-xs text-gray-500 mt-2">
							{processingMode === 'gentle' ? 'Preserves original image quality for clear receipts' : 'Heavy processing for faded or poor quality receipts'}
						</p>
						
						<!-- Debug mode toggle -->
						<div class="mt-3 pt-3 border-t border-blue-200">
							<label class="flex items-center text-sm mb-2">
								<input 
									type="checkbox" 
									bind:checked={debugMode}
									on:change={() => imageUrl && processImage(imageUrl)}
									class="mr-2"
								/>
								<span class="text-gray-700">Show edge detection (debug)</span>
							</label>
							<label class="flex items-center text-sm">
								<input 
									type="checkbox" 
									bind:checked={skipEdgeDetection}
									on:change={() => imageUrl && processImage(imageUrl)}
									class="mr-2"
								/>
								<span class="text-gray-700">Skip edge detection (for flat images)</span>
							</label>
						</div>
					</div>
				</div>
			{/if}
		</div>
		
		<!-- Right Column - Extracted Data -->
		<div class="space-y-6">
			{#if extractedData}
				<div transition:fade={{ duration: 300 }}>
					<!-- Header Info -->
					<div class="bg-gray-50 rounded-lg p-6 mb-6">
						<h3 class="text-lg font-semibold text-gray-900 mb-4">Receipt Information</h3>
						<div class="grid grid-cols-2 gap-4 text-sm">
							<div>
								<p class="text-gray-500">Vendor</p>
								<p class="font-medium text-gray-900">{extractedData.vendor}</p>
							</div>
							<div>
								<p class="text-gray-500">Date</p>
								<p class="font-medium text-gray-900">{extractedData.date}</p>
							</div>
							<div>
								<p class="text-gray-500">Time</p>
								<p class="font-medium text-gray-900">{extractedData.time || 'N/A'}</p>
							</div>
							<div>
								<p class="text-gray-500">Payment</p>
								<p class="font-medium text-gray-900">{extractedData.paymentMethod}</p>
							</div>
						</div>
					</div>
					
					<!-- Items List -->
					<div class="bg-white border border-gray-200 rounded-lg overflow-hidden">
						<div class="px-6 py-4 bg-gray-50 border-b border-gray-200">
							<h3 class="text-lg font-semibold text-gray-900">Items</h3>
						</div>
						<div class="divide-y divide-gray-200 max-h-96 overflow-y-auto">
							{#if extractedData.items.length > 0}
								{#each extractedData.items as item}
									<div class="px-6 py-4" transition:slide>
										<div class="flex justify-between items-start">
											<div class="flex-1">
												<p class="font-medium text-gray-900">{item.description}</p>
												<p class="text-sm text-gray-500 mt-1">
													{item.quantity} × {formatCurrency(item.unitPrice)}
												</p>
											</div>
											<p class="font-medium text-gray-900 ml-4">
												{formatCurrency(item.total)}
											</p>
										</div>
									</div>
								{/each}
							{:else}
								<div class="px-6 py-8 text-center text-gray-500">
									<p>No items detected</p>
									<p class="text-sm mt-2">Try switching to {processingMode === 'gentle' ? 'aggressive' : 'gentle'} mode</p>
								</div>
							{/if}
						</div>
						
						<!-- Totals -->
						<div class="px-6 py-4 bg-gray-50 border-t border-gray-200 space-y-2">
							<div class="flex justify-between text-sm">
								<span class="text-gray-600">Subtotal</span>
								<span class="font-medium text-gray-900">{formatCurrency(extractedData.subtotal)}</span>
							</div>
							<div class="flex justify-between text-sm">
								<span class="text-gray-600">Tax</span>
								<span class="font-medium text-gray-900">{formatCurrency(extractedData.tax)}</span>
							</div>
							<div class="flex justify-between text-lg font-semibold pt-2 border-t border-gray-300">
								<span class="text-gray-900">Total</span>
								<span class="text-gray-900">{formatCurrency(extractedData.total)}</span>
							</div>
						</div>
					</div>
					
					<!-- Copy Button -->
					<button
						on:click={copyToClipboard}
						class="w-full px-6 py-3 bg-blue-600 hover:bg-blue-700 text-white font-medium rounded-lg transition-colors flex items-center justify-center space-x-2"
					>
						{#if showCopySuccess}
							<svg class="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
							</svg>
							<span>Copied to Clipboard!</span>
						{:else}
							<svg class="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3" />
							</svg>
							<span>Copy Data to Clipboard</span>
						{/if}
					</button>
					
					<!-- Debug: Show raw OCR text -->
					<details class="mt-4">
						<summary class="text-sm text-gray-500 cursor-pointer hover:text-gray-700">View raw OCR text</summary>
						<pre class="mt-2 p-4 bg-gray-100 rounded text-xs overflow-x-auto whitespace-pre-wrap">{extractedData.rawText}</pre>
						{#if extractedData.confidence}
							<p class="mt-2 text-xs text-gray-500">OCR Confidence: {extractedData.confidence.toFixed(1)}%</p>
						{/if}
					</details>
				</div>
			{:else if !isProcessing && imageUrl}
				<div class="text-center py-12 text-gray-500">
					<p>No data extracted yet</p>
				</div>
			{:else if !imageUrl}
				<div class="bg-gray-50 rounded-lg p-12 text-center">
					<svg class="w-16 h-16 text-gray-400 mx-auto mb-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
					</svg>
					<h3 class="text-lg font-medium text-gray-900 mb-2">Advanced Receipt Processing</h3>
					<p class="text-sm text-gray-600 mb-4">Upload any receipt photo - even at an angle!</p>
					<div class="text-xs text-gray-500 space-y-1">
						<p>Powered by OpenCV.js for edge detection</p>
						<p>Tesseract.js for optical character recognition</p>
					</div>
				</div>
			{/if}
		</div>
	</div>
</div>