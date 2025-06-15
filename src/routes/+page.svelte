<script lang="ts">
	import { fade, fly, scale } from 'svelte/transition';
	import { quintOut } from 'svelte/easing';
	import ReceiptScanner from '$lib/ReceiptScanner.svelte';
	import HomeInspectionForm from '$lib/HomeInspectionForm.svelte';
	import StepFileViewer from '$lib/StepFileViewer.svelte';
	
	// Contact form handling
	let formData = {
		name: '',
		email: '',
		message: ''
	};

	let isSubmitting = false;
	let showSuccess = false;
	let showError = false;

	async function handleFormspreeSubmit(event) {
		event.preventDefault();
		isSubmitting = true;
		showError = false;
		showSuccess = false;

		try {
			const form = event.target;
			const formDataToSubmit = new FormData(form);
			const response = await fetch(form.action, {
				method: 'POST',
				body: formDataToSubmit,
				headers: {
					'Accept': 'application/json'
				}
			});

			if (response.ok) {
				showSuccess = true;
				// Reset form fields individually
				formData.name = '';
				formData.email = '';
				formData.message = '';
				
				// Also reset the actual form element
				form.reset();
				
				// Hide success message after 5 seconds
				setTimeout(() => {
					showSuccess = false;
				}, 5000);
			} else {
				throw new Error('Form submission failed');
			}
		} catch (error) {
			console.error('Form submission error:', error);
			showError = true;
			
			// Hide error message after 5 seconds
			setTimeout(() => {
				showError = false;
			}, 5000);
		} finally {
			isSubmitting = false;
		}
	}

	// Project expansion handling
	let expandedProject: string | null = null;
	
	function expandProject(projectId: string) {
		expandedProject = projectId;
		// Prevent body scroll when modal is open
		document.body.style.overflow = 'hidden';
	}
	
	function closeProject() {
		expandedProject = null;
		// Re-enable body scroll
		document.body.style.overflow = 'auto';
	}
	
	// Handle escape key
	function handleKeydown(event: KeyboardEvent) {
		if (event.key === 'Escape' && expandedProject) {
			closeProject();
		}
	}
</script>

<svelte:window on:keydown={handleKeydown} />

<svelte:head>
	<title>Allen Flett, AScT - Giants Head Machinery and Design</title>
	<meta name="description" content="Construction management software and mechanical engineering solutions focused on user ergonomics and practical functionality" />
</svelte:head>

<!-- Hero Section -->
<section id="hero" class="py-4 lg:py-16">
	<div class="flex flex-col lg:flex-row items-start justify-center gap-0">
		<!-- Hero Content -->
		<div class="w-full lg:w-auto text-center lg:text-left">
			<h1 class="text-4xl lg:text-5xl font-bold text-gray-900 handwriting mb-6">
				Modern Tools<br>
				<span class="text-logo-green">Working for you</span>
			</h1>
			<p class="text-xl text-gray-700 leading-relaxed mb-8 max-w-2xl mx-auto lg:mx-0">
				I build practical construction management applications focused on your processes and individual business needs. My tools are convenient, adaptable, and designed to reduce 
				overhead while increasing outputs.
			</p>
			<div class="flex flex-col sm:flex-row gap-4 justify-center lg:justify-start">
				<a 
					href="#projects" 
					class="px-8 py-3 logo-green text-white font-medium rounded-lg transition-colors duration-200 handwriting"
				>
					View My Work
				</a>
				<a 
					href="#contact" 
					class="px-8 py-3 border-2 border-gray-300 hover:border-logo-teal text-gray-700 hover:text-logo-teal font-medium rounded-lg transition-colors duration-200 handwriting"
				>
					Get In Touch
				</a>
			</div>
		</div>

		<!-- Profile Photo - Hidden on mobile/tablet, visible on large screens -->
		<div class="hidden lg:block flex-shrink-0">
			<div class="w-96">
				<!-- Hand-sketched profile photo with transparent background -->
				<img 
					src="/profile-sketch.png" 
					alt="Allen Flett - Hand-sketched portrait" 
					class="w-full h-auto"
					style="filter: contrast(1.1) brightness(1.05); margin-top: -6rem;"
				/>
			</div>
		</div>
	</div>
</section>

<!-- About Section -->
<section id="about" class="py-12">
	<div class="space-y-8">
		<h2 class="text-3xl font-bold text-gray-900 handwriting">About Allen</h2>
		<div class="prose prose-lg max-w-none space-y-4">
			<p class="text-gray-700 text-lg leading-relaxed">
				I'm a Mechanical Technologist and Principal at Giants Head Machinery and Design, specializing in 
				construction management software that actually works for the people who use it. My core philosophy: make a 
				product for your company, not fit your company into my product.
			</p>
			<p class="text-gray-700 text-lg leading-relaxed">
				My approach to construction management apps prioritizes <strong>user ergonomics for easy employee buy-in</strong>. 
				I build tools that are convenient, adaptable, supporting, and complimentary - reducing overhead 
				while increasing outputs. Whether it's tool tracking, purchase orders, or a fully customised construction management 
				software (CMS), I focus on cognitive ergonomics and accessibility across mobile devices, 
				laptops, and desktop platforms.
			</p>
			<p class="text-gray-700 text-lg leading-relaxed">
				The goal isn't just over simplified user experiences or a my way or the hiway approach - it's practical solutions that are built specifically for your business. I want to eliminate the need to 
				chase contractors or scroll through emails looking for information. I automate simple tasks, 
				add missing secondary processes like JLHA and QC, and optimize workflows to reduce tribal knowledge.
			</p>
		</div>
	</div>
</section>

<!-- Projects Section -->
<section id="projects" class="py-12">
	<div class="space-y-8">
		<h2 class="text-3xl font-bold text-gray-900 handwriting">Interactive Tool Demonstrations</h2>
		<div class="grid grid-cols-1 md:grid-cols-3 gap-8">
			<!-- Receipt Scanner -->
			<button
				on:click={() => expandProject('receipt-scanner')}
				class="border-2 border-gray-300 rounded-lg p-6 hover:border-logo-teal transition-all duration-300 cursor-pointer text-left group relative overflow-hidden hover:shadow-lg"
			>
				<div class="relative z-10">
					<h3 class="text-xl font-semibold text-gray-900 handwriting mb-4">Receipt Scanner - In Training</h3>
					<div class="mb-4">
						<!-- Placeholder for sketch image -->
						<img 
							src="/reciept.png" 
							alt="reciept - Hand-sketched" 
							class="w-auto h-45"
							style="filter: contrast(1.1) brightness(1.05); "
						/>
					</div>
					<p class="text-gray-600 mb-3 text-sm">Extract and organize expense data from receipts using OCR technology. Automatic categorization and export capabilities. Finished API's available for faster development.</p>
					<div class="text-xs text-gray-500 italic">
						Click to try the demo →
					</div>
				</div>
				<div class="absolute inset-0 bg-gradient-to-br from-transparent to-green-50 opacity-0 group-hover:opacity-100 transition-opacity duration-300"></div>
			</button>

			<!-- AI Document Chat -->
			<button
				on:click={() => expandProject('ai-chat')}
				class="border-2 border-gray-300 rounded-lg p-6 hover:border-logo-teal transition-all duration-300 cursor-pointer text-left group relative overflow-hidden hover:shadow-lg"
			>
				<div class="relative z-10">
					<h3 class="text-xl font-semibold text-gray-900 handwriting mb-4">AI Document Assistant - In Training</h3>
					<div class="mb-4">
						<!-- Placeholder for sketch image -->
						<img 
							src="/chat.png" 
							alt="chatbot - Hand-sketched" 
							class="w-auto h-45"
							style="filter: contrast(1.1) brightness(1.05);"
						/>
					</div>
					<p class="text-gray-600 mb-3 text-sm">Chat with your technical documents. Get instant answers from manuals, specifications, and company procedures.</p>
					<div class="text-xs text-gray-500 italic">
						Click to try the demo →
					</div>
				</div>
				<div class="absolute inset-0 bg-gradient-to-br from-transparent to-blue-50 opacity-0 group-hover:opacity-100 transition-opacity duration-300"></div>
			</button>

			<!-- Third Tool (Digital Forms) -->
			<button
				on:click={() => expandProject('tool-three')}
				class="border-2 border-gray-300 rounded-lg p-6 hover:border-logo-teal transition-all duration-300 cursor-pointer text-left group relative overflow-hidden hover:shadow-lg"
			>
				<div class="relative z-10">
					<h3 class="text-xl font-semibold text-gray-900 handwriting mb-4">Custom Digital Forms</h3>
					<div class="mb-4">
						<!-- Placeholder for sketch image -->
						<img 
							src="/form.png" 
							alt="form - Hand-sketched" 
							class="w-auto h-45"
							style="filter: contrast(1.1) brightness(1.05); "
						/>
					</div>
					<p class="text-gray-600 mb-3 text-sm">Convert your paper forms to digital. Mobile-friendly interface for field workers with offline support.</p>
					<div class="text-xs text-gray-500 italic">
						Click to explore →
					</div>
				</div>
				<div class="absolute inset-0 bg-gradient-to-br from-transparent to-purple-50 opacity-0 group-hover:opacity-100 transition-opacity duration-300"></div>
			</button>
		</div>
		
		<!-- Second Row with Fourth Tool -->
		<div class="grid grid-cols-1 md:grid-cols-3 gap-8">
			<!-- STEP File Viewer -->
			<button
				on:click={() => expandProject('step-viewer')}
				class="border-2 border-gray-300 rounded-lg p-6 hover:border-logo-teal transition-all duration-300 cursor-pointer text-left group relative overflow-hidden hover:shadow-lg"
			>
				<div class="relative z-10">
					<h3 class="text-xl font-semibold text-gray-900 handwriting mb-4">3D STEP File Viewer</h3>
					<div class="mb-4">
						<!-- 3D icon representation -->
						<img 
							src="/CAD.png" 
							alt="cad file - Hand-sketched" 
							class="w-auto h-45"
							style="filter: contrast(1.1) brightness(1.05); "
						/>
					</div>
					<p class="text-gray-600 mb-3 text-sm">View and rotate 3D CAD models directly in your browser. Support for STEP files from any CAD software.</p>
					<div class="text-xs text-gray-500 italic">
						Click to view demo →
					</div>
				</div>
				<div class="absolute inset-0 bg-gradient-to-br from-transparent to-emerald-50 opacity-0 group-hover:opacity-100 transition-opacity duration-300"></div>
			</button>
		</div>
		
		<!-- Philosophy Section -->
		<div class="mt-12 p-6 bg-gray-50 rounded-lg border-2 border-gray-200">
			<h3 class="text-xl font-semibold text-gray-900 handwriting mb-4">Design Philosophy</h3>
			<div class="grid grid-cols-1 md:grid-cols-4 gap-6 text-center">
				<div>
					<div class="text-2xl font-bold text-logo-teal handwriting">01</div>
					<h4 class="font-semibold text-gray-900 handwriting">Convenient</h4>
					<p class="text-sm text-gray-600">Access and ease of use for field and management staff</p>
				</div>
				<div>
					<div class="text-2xl font-bold text-logo-teal handwriting">02</div>
					<h4 class="font-semibold text-gray-900 handwriting">Adaptable</h4>
					<p class="text-sm text-gray-600">Optimized and customized for your company's workflow</p>
				</div>
				<div>
					<div class="text-2xl font-bold text-logo-teal handwriting">03</div>
					<h4 class="font-semibold text-gray-900 handwriting">Supporting</h4>
					<p class="text-sm text-gray-600">Automation of simple tasks and reporting</p>
				</div>
				<div>
					<div class="text-2xl font-bold text-logo-teal handwriting">04</div>
					<h4 class="font-semibold text-gray-900 handwriting">Complimentary</h4>
					<p class="text-sm text-gray-600">Enhancing operations without burdening resources</p>
				</div>
			</div>
		</div>
	</div>
</section>

<!-- Expanded Project Modal -->
{#if expandedProject}
	<div 
		class="fixed inset-0 z-50 flex items-center justify-center p-4"
		transition:fade={{ duration: 200 }}
		on:click={closeProject}
		on:keydown={() => {}}
		role="button"
		tabindex="0"
	>
		<!-- Backdrop -->
		<div class="absolute inset-0 bg-black bg-opacity-50"></div>
		
		<!-- Modal Content -->
		<div 
			class="relative bg-white rounded-xl shadow-2xl w-full max-w-6xl max-h-[90vh] overflow-hidden"
			transition:scale={{ duration: 300, easing: quintOut, start: 0.9 }}
			on:click|stopPropagation
			on:keydown={() => {}}
			role="button"
			tabindex="0"
		>
			<!-- Close Button -->
			<button
				on:click={closeProject}
				class="absolute top-4 right-4 z-10 p-2 bg-white rounded-full shadow-lg hover:bg-gray-100 transition-colors"
				aria-label="Close"
			>
				<svg class="w-6 h-6 text-gray-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
				</svg>
			</button>
			
			<!-- Tool Content Area -->
			<div class="p-8 max-h-[85vh] overflow-y-auto" data-app="true">
				{#if expandedProject === 'receipt-scanner'}
					<ReceiptScanner />
				{:else if expandedProject === 'ai-chat'}
					<div class="text-center py-16">
						<h2 class="text-3xl font-bold text-gray-900 mb-4">AI Document Assistant</h2>
						<p class="text-gray-600 mb-8">Chat with your technical documents for instant answers</p>
						<div class="inline-flex items-center justify-center w-64 h-64 bg-gray-100 rounded-lg">
							<svg class="w-24 h-24 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M8 10h.01M12 10h.01M16 10h.01M9 16H5a2 2 0 01-2-2V6a2 2 0 012-2h14a2 2 0 012 2v8a2 2 0 01-2 2h-5l-5 5v-5z" />
							</svg>
						</div>
						<p class="mt-4 text-sm text-gray-500">Tool implementation coming soon...</p>
					</div>
				{:else if expandedProject === 'tool-three'}
					<HomeInspectionForm />
				{:else if expandedProject === 'step-viewer'}
					<StepFileViewer />
				{/if}
			</div>
		</div>
	</div>
{/if}

<!-- Contact Section -->
<section id="contact" class="py-12">
	<div class="space-y-8">
		<h2 class="text-3xl font-bold text-gray-900 handwriting">Get In Touch</h2>
		
		<div class="grid grid-cols-1 lg:grid-cols-2 gap-12">
			<!-- Contact Form -->
			<div>
				<form 
					action="https://formspree.io/f/myzjjzya" 
					method="POST"
					on:submit={handleFormspreeSubmit}
					class="space-y-6"
				>
					<div>
						<label for="name" class="block text-sm font-medium text-gray-900 mb-2 handwriting">
							Name *
						</label>
						<input
							type="text"
							id="name"
							name="name"
							bind:value={formData.name}
							required
							class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-logo-teal focus:border-transparent"
							placeholder="Your name"
						/>
					</div>

					<div>
						<label for="email" class="block text-sm font-medium text-gray-900 mb-2 handwriting">
							Email *
						</label>
						<input
							type="email"
							id="email"
							name="email"
							bind:value={formData.email}
							required
							class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-logo-teal focus:border-transparent"
							placeholder="your@email.com"
						/>
					</div>

					<div>
						<label for="message" class="block text-sm font-medium text-gray-900 mb-2 handwriting">
							Message *
						</label>
						<textarea
							id="message"
							name="message"
							bind:value={formData.message}
							required
							rows="5"
							class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-logo-teal focus:border-transparent resize-vertical"
							placeholder="Tell me about your construction management software needs..."
						></textarea>
					</div>

					<!-- Hidden fields for Formspree -->
					<input type="hidden" name="_subject" value="New contact from portfolio website" />

					{#if showSuccess}
						<div class="p-4 bg-green-50 border border-green-200 rounded-lg">
							<p class="text-green-800 font-medium handwriting">
								✓ Message sent successfully! I'll get back to you soon.
							</p>
						</div>
					{/if}

					{#if showError}
						<div class="p-4 bg-red-50 border border-red-200 rounded-lg">
							<p class="text-red-800 font-medium handwriting">
								✗ Failed to send message. Please try again or email me directly.
							</p>
						</div>
					{/if}

					<button
						type="submit"
						disabled={isSubmitting}
						class="w-full px-6 py-3 bg-logo-green hover:bg-opacity-90 disabled:bg-gray-400 disabled:cursor-not-allowed text-white font-medium rounded-lg transition-colors duration-200 flex items-center justify-center space-x-2 handwriting"
					>
						{#if isSubmitting}
							<div class="w-4 h-4 border-2 border-white border-t-transparent rounded-full animate-spin"></div>
							<span>Sending...</span>
						{:else}
							<span>Send Message</span>
						{/if}
					</button>
				</form>
			</div>
		</div>
	</div>
</section>

<!-- Footer -->
<footer class="py-8 border-t border-gray-200 mt-12">
	<div class="text-center text-sm text-gray-600 handwriting">
		<p>© 2024 Allen Flett - Giants Head Machinery and Design. All rights reserved.</p>
	</div>
</footer>