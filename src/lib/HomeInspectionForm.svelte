<script lang="ts">
	import { fade, fly } from 'svelte/transition';
	import { quintOut } from 'svelte/easing';
	
	let formData = {
		inspectionDate: '',
		propertyType: '',
		images: [] as File[],
		notes: '',
		priority: 'standard',
		completionLevel: 70,
		areas: [] as string[],
		timeSlot: '',
		rating: 0
	};
	
	let showSuccess = false;
	let imagePreview: string | null = null;
	let fileInput: HTMLInputElement;
	let hoveredRating = 0;

	// Areas for checkbox selection
	const inspectionAreas = [
		{ id: 'foundation', label: 'Foundation' },
		{ id: 'roof', label: 'Roof' },
		{ id: 'plumbing', label: 'Plumbing' },
		{ id: 'electrical', label: 'Electrical' },
		{ id: 'hvac', label: 'HVAC' },
		{ id: 'interior', label: 'Interior' },
		{ id: 'exterior', label: 'Exterior' },
		{ id: 'garage', label: 'Garage' }
	];

	// Time slots
	const timeSlots = [
		{ value: 'morning', label: '8:00 AM - 12:00 PM' },
		{ value: 'afternoon', label: '12:00 PM - 4:00 PM' },
		{ value: 'evening', label: '4:00 PM - 7:00 PM' }
	];

	function handleImageUpload(event: Event) {
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		if (file) {
			const reader = new FileReader();
			reader.onload = (e) => {
				imagePreview = e.target?.result as string;
				formData.images = [file];
			};
			reader.readAsDataURL(file);
		}
	}

	function toggleArea(areaId: string) {
		if (formData.areas.includes(areaId)) {
			formData.areas = formData.areas.filter(id => id !== areaId);
		} else {
			formData.areas = [...formData.areas, areaId];
		}
	}

	function handleSubmit() {
		showSuccess = true;
		
		// Reset after 3 seconds
		setTimeout(() => {
			showSuccess = false;
			formData = {
				inspectionDate: '',
				propertyType: '',
				images: [],
				notes: '',
				priority: 'standard',
				completionLevel: 70,
				areas: [],
				timeSlot: '',
				rating: 0
			};
			imagePreview = null;
			hoveredRating = 0;
			if (fileInput) fileInput.value = '';
		}, 3000);
	}

	function getCurrentDateTime() {
		return new Date().toLocaleString('en-US', {
			weekday: 'long',
			year: 'numeric',
			month: 'long',
			day: 'numeric',
			hour: '2-digit',
			minute: '2-digit'
		});
	}

	function getCompletionColor(value: number) {
		if (value < 30) return 'bg-red-500';
		if (value < 60) return 'bg-yellow-500';
		return 'bg-logo-green';
	}

	$: isFormValid = formData.inspectionDate && formData.propertyType && formData.areas.length > 0 && formData.timeSlot;
	$: completionColor = getCompletionColor(formData.completionLevel);
</script>

<!-- Main Container with Grey Background -->
<div class="min-h-screen bg-gray-100 p-4">
	{#if showSuccess}
		<div class="max-w-md mx-auto" transition:fade>
			<!-- Success Card -->
			<div class="bg-white rounded-2xl shadow-lg overflow-hidden">
				<div class="bg-logo-green p-8 text-center">
					<div class="w-20 h-20 bg-white rounded-full flex items-center justify-center mx-auto mb-4">
						<svg class="w-10 h-10 text-logo-green" fill="none" viewBox="0 0 24 24" stroke="currentColor">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="3" d="M5 13l4 4L19 7" />
						</svg>
					</div>
					<h2 class="text-2xl font-bold text-white mb-2 handwriting">Inspection Scheduled!</h2>
					<p class="text-green-100 text-sm body-font">
						{getCurrentDateTime()}
					</p>
				</div>
				
				<div class="p-6">
					<div class="space-y-3">
						<div class="flex items-center justify-between p-3 bg-gray-50 rounded-lg">
							<span class="text-gray-600 text-sm body-font">Confirmation #</span>
							<span class="font-mono text-sm font-bold text-gray-900">INS-{Math.floor(Math.random() * 9000) + 1000}</span>
						</div>
						
						<div class="flex items-center justify-between p-3 bg-gray-50 rounded-lg">
							<span class="text-gray-600 text-sm body-font">Duration</span>
							<span class="font-semibold text-gray-900 text-sm body-font">2-3 Hours</span>
						</div>
						
						<div class="text-center pt-4">
							<p class="text-xs text-gray-500 body-font">
								Confirmation email sent to your address
							</p>
						</div>
					</div>
				</div>
			</div>
		</div>
	{:else}
		<div class="max-w-4xl mx-auto">
			<!-- Main Form Card -->
			<div class="bg-white rounded-2xl shadow-xl overflow-hidden">
				<!-- Header -->
				<div class="bg-logo-green p-6 text-center">
					<h2 class="text-2xl font-bold text-white mb-2 handwriting">Home Inspection Booking</h2>
					<p class="text-green-100 text-sm body-font">Professional inspection services</p>
				</div>

				<!-- Form Body -->
				<div class="p-6">
					<div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
						<!-- Left Column -->
						<div class="space-y-6">
							<!-- Date Selection -->
							<div>
								<label class="block text-sm font-medium text-gray-700 mb-2 body-font">Inspection Date</label>
								<input
									type="date"
									bind:value={formData.inspectionDate}
									class="w-full px-4 py-3 bg-white border border-gray-300 rounded-lg focus:border-logo-green focus:ring-1 focus:ring-logo-green transition-all text-gray-700"
									required
								/>
							</div>

							<!-- Time Slots -->
							<div>
								<label class="block text-sm font-medium text-gray-700 mb-3 body-font">Time Slot</label>
								<div class="space-y-2">
									{#each timeSlots as slot}
										<label class="block">
											<input
												type="radio"
												bind:group={formData.timeSlot}
												value={slot.value}
												class="sr-only"
											/>
											<div class="p-3 rounded-lg border cursor-pointer transition-all {
												formData.timeSlot === slot.value 
													? 'border-logo-green bg-green-50' 
													: 'border-gray-300 bg-white hover:border-gray-400'
											}">
												<p class="font-medium text-gray-900 text-sm">{slot.label}</p>
											</div>
										</label>
									{/each}
								</div>
							</div>

							<!-- Property Type -->
							<div>
								<label class="block text-sm font-medium text-gray-700 mb-3 body-font">Property Type</label>
								<div class="grid grid-cols-2 gap-3">
									{#each [
										{ value: 'single-family', label: 'Single Family' },
										{ value: 'condo', label: 'Condo' },
										{ value: 'townhouse', label: 'Townhouse' },
										{ value: 'apartment', label: 'Apartment' }
									] as option}
										<label class="block">
											<input
												type="radio"
												bind:group={formData.propertyType}
												value={option.value}
												class="sr-only"
											/>
											<div class="p-4 text-center rounded-lg border cursor-pointer transition-all {
												formData.propertyType === option.value 
													? 'border-logo-green bg-green-50' 
													: 'border-gray-300 bg-white hover:border-gray-400'
											}">
												<span class="font-medium text-gray-900 text-sm body-font">{option.label}</span>
											</div>
										</label>
									{/each}
								</div>
							</div>

							<!-- Photo Upload -->
							<div>
								<label class="block text-sm font-medium text-gray-700 mb-2 body-font">Property Photo</label>
								<input
									type="file"
									accept="image/*"
									on:change={handleImageUpload}
									bind:this={fileInput}
									class="sr-only"
									id="property-photo"
								/>
								<label
									for="property-photo"
									class="flex items-center justify-center w-full h-32 border-2 border-dashed border-gray-300 rounded-lg cursor-pointer bg-gray-50 hover:bg-gray-100 transition-all"
								>
									{#if imagePreview}
										<img 
											src={imagePreview} 
											alt="Property preview" 
											class="w-full h-full object-cover rounded-lg"
										/>
									{:else}
										<div class="text-center">
											<svg class="w-8 h-8 text-gray-400 mx-auto mb-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
												<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
											</svg>
											<p class="text-xs text-gray-500">Upload photo</p>
										</div>
									{/if}
								</label>
							</div>
						</div>

						<!-- Right Column -->
						<div class="space-y-6">
							<!-- Inspection Areas -->
							<div>
								<label class="block text-sm font-medium text-gray-700 mb-3 body-font">Areas to Inspect</label>
								<div class="grid grid-cols-2 gap-2">
									{#each inspectionAreas as area}
										<label class="flex items-center p-3 rounded-lg border cursor-pointer transition-all {
											formData.areas.includes(area.id)
												? 'border-logo-green bg-green-50'
												: 'border-gray-300 bg-white hover:border-gray-400'
										}">
											<input
												type="checkbox"
												checked={formData.areas.includes(area.id)}
												on:change={() => toggleArea(area.id)}
												class="sr-only"
											/>
											<div class="flex items-center space-x-2">
												<div class="w-4 h-4 rounded border-2 flex items-center justify-center {
													formData.areas.includes(area.id)
														? 'border-logo-green bg-logo-green'
														: 'border-gray-400 bg-white'
												}">
													{#if formData.areas.includes(area.id)}
														<svg class="w-3 h-3 text-white" fill="currentColor" viewBox="0 0 20 20">
															<path fill-rule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clip-rule="evenodd" />
														</svg>
													{/if}
												</div>
												<span class="text-sm text-gray-700">{area.label}</span>
											</div>
										</label>
									{/each}
								</div>
							</div>

							<!-- Urgency Slider -->
							<div>
								<label class="block text-sm font-medium text-gray-700 mb-3 body-font">Urgency Level</label>
								<div class="bg-gray-50 rounded-lg p-4">
									<input
										type="range"
										bind:value={formData.completionLevel}
										min="0"
										max="100"
										step="10"
										class="w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer slider"
									/>
									<div class="flex justify-between text-xs text-gray-500 mt-2">
										<span>Low</span>
										<span>Medium</span>
										<span>High</span>
									</div>
									<div class="text-center mt-3">
										<span class="text-lg font-bold {completionColor} bg-clip-text text-transparent">{formData.completionLevel}%</span>
									</div>
								</div>
							</div>

							<!-- Notes -->
							<div>
								<label class="block text-sm font-medium text-gray-700 mb-2 body-font">Additional Notes</label>
								<textarea
									bind:value={formData.notes}
									placeholder="Special instructions..."
									rows="4"
									class="w-full px-4 py-3 bg-white border border-gray-300 rounded-lg focus:border-logo-green focus:ring-1 focus:ring-logo-green transition-all resize-none text-gray-700 text-sm"
								></textarea>
							</div>
						</div>
					</div>

					<!-- Submit Button -->
					<div class="mt-6 pt-6 border-t border-gray-200">
						<button
							on:click={handleSubmit}
							disabled={!isFormValid}
							class="w-full py-3 bg-logo-green text-white font-medium rounded-lg transition-all duration-200 hover:bg-opacity-90 disabled:bg-gray-300 disabled:cursor-not-allowed shadow-sm hover:shadow-md handwriting"
						>
							{#if !isFormValid}
								Complete Required Fields
							{:else}
								Submit Inspection Request
							{/if}
						</button>
					</div>
				</div>
			</div>
		</div>
	{/if}
</div>

<style>
	/* Custom slider styling */
	.slider {
		background: linear-gradient(to right, #ef4444 0%, #eab308 50%, #2d5a2d 100%);
	}
	
	.slider::-webkit-slider-thumb {
		appearance: none;
		width: 20px;
		height: 20px;
		background: white;
		border: 2px solid #d1d5db;
		border-radius: 50%;
		cursor: pointer;
		box-shadow: 0 1px 3px rgba(0,0,0,0.1);
	}
	
	.slider::-moz-range-thumb {
		width: 20px;
		height: 20px;
		background: white;
		border: 2px solid #d1d5db;
		border-radius: 50%;
		cursor: pointer;
		box-shadow: 0 1px 3px rgba(0,0,0,0.1);
	}
</style>