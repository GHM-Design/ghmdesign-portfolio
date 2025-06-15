<script lang="ts">
	import '../app.css';
	import { onMount } from 'svelte';

	// Navigation items
	const navItems = [
		{ href: '#about', label: 'ABOUT' },
		{ href: '#projects', label: 'PROJECTS' },
		{ href: '#contact', label: 'CONTACT' }
	];

	let activeSection = 'about';
	let sideMenuOpen = false;
	let windowWidth = 1400; // Default to show header

	// Smooth scroll function
	function scrollToSection(sectionId: string) {
		const element = document.getElementById(sectionId);
		if (element) {
			element.scrollIntoView({ 
				behavior: 'smooth',
				block: 'start'
			});
		}
		// Close side menu after navigation
		sideMenuOpen = false;
	}

	// Toggle side menu
	function toggleSideMenu() {
		sideMenuOpen = !sideMenuOpen;
	}

	// Track active section on scroll
	onMount(() => {
		// Handle window resize
		const handleResize = () => {
			windowWidth = window.innerWidth;
		};
		
		handleResize(); // Set initial width
		window.addEventListener('resize', handleResize);

		const observerOptions = {
			root: null,
			rootMargin: '-50% 0px -50% 0px',
			threshold: 0
		};

		const observer = new IntersectionObserver((entries) => {
			entries.forEach((entry) => {
				if (entry.isIntersecting) {
					activeSection = entry.target.id;
				}
			});
		}, observerOptions);

		// Observe all sections
		const sections = document.querySelectorAll('section[id]');
		sections.forEach((section) => observer.observe(section));

		// Animate menu items on load (only for desktop)
		if (windowWidth > 1300) {
			const menuLinks = document.querySelectorAll('.menu-link');
			menuLinks.forEach((link, index) => {
				setTimeout(() => {
					link.classList.add('highlight-animation');
					setTimeout(() => {
						link.classList.remove('highlight-animation');
					}, 500);
				}, index * 600);
			});
		}

		return () => {
			observer.disconnect();
			window.removeEventListener('resize', handleResize);
		};
	});

	// Menu items for side navigation
	const menuItems = [
		{ icon: 'user', label: 'About', href: '#about' },
		{ icon: 'tools', label: 'Tool Demos', href: '#projects' },
		{ icon: 'mail', label: 'Contact', href: '#contact' }
	];
</script>

<svelte:head>
	<title>Allen Flett, AScT - Giants Head Machinery and Design</title>
	<meta name="description" content="Construction management software and mechanical engineering solutions focused on user ergonomics and practical functionality" />
	<link rel="preconnect" href="https://fonts.googleapis.com">
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
	<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700;800;900&family=Kalam:wght@400;700&family=Roboto+Mono:wght@300;400;500;600;700&family=Coming+Soon&display=swap" rel="stylesheet">
	
	<!-- Tailwind CDN with font configuration -->
	<script src="https://cdn.tailwindcss.com"></script>
	<script>
		tailwind.config = {
			theme: {
				extend: {
					fontFamily: {
						'coming-soon': ['Coming Soon', 'cursive'],
						'roboto-mono': ['Roboto Mono', 'monospace'],
						'kalam': ['Kalam', 'cursive']
					}
				}
			}
		}
	</script>
</svelte:head>

{#if windowWidth > 1300}
	<!-- Desktop Header in white margin area -->
	<div class="header-margin">
		<div class="flex justify-between items-end w-full h-full pb-4">
			<!-- Logo and Company Info on Left -->
			<div class="flex items-end space-x-4">
				<div class="flex-shrink-0 w-60 h-30">
					<img 
						src="/giants-head-logo.png" 
						alt="Giants Head Machinery and Design Logo" 
						class="w-full h-full object-contain"
					/>
				</div>
				<div class="text-sm text-gray-700 leading-tight header-font pb-1">
					<h1 class="text-logo-green text-base font-semibold">GIANTS HEAD MACHINERY AND DESIGN</h1>
					<p>Summerland, BC</p>
					<p>ALLEN FLETT, AScT</p>
					<p>PRINCIPAL & MECHANICAL TECHNOLOGIST</p>
				</div>
			</div>

			<!-- Engineering Header Menu on Right - No Box -->
			<div class="text-sm header-font leading-tight w-full max-w-xl pb-1">
				<div class="space-y-1">
					<div class="flex items-center justify-between">
						<span class="text-gray-700">PROJECT</span>
						<span class="handwritten-underline flex-1 text-center ml-4">PORTFOLIO SITE</span>
					</div>
					<div class="flex items-center justify-between">
						<span class="text-gray-700">FOR</span>
						<a href="#about" on:click|preventDefault={() => scrollToSection('about')} class="handwritten-underline menu-link flex-1 text-center ml-4 hover:text-logo-green hover:bg-green-50 cursor-pointer transition-all duration-200">ALLEN FLETT AScT</a>
					</div>
					<div class="flex items-center justify-between">
						<span class="text-gray-700">APPS</span>
						<a href="#projects" on:click|preventDefault={() => scrollToSection('projects')} class="handwritten-underline menu-link flex-1 text-center ml-4 hover:text-logo-green hover:bg-green-50 cursor-pointer transition-all duration-200">CMS TOOLS</a>
					</div>
					<div class="flex items-center justify-between">
						<span class="text-gray-700">CONTACT</span>
						<a href="#contact" on:click|preventDefault={() => scrollToSection('contact')} class="flex-1 ml-4 flex items-center justify-center space-x-2 menu-link hover:text-logo-green hover:bg-green-50 cursor-pointer transition-all duration-200 rounded-md">
							<span class="handwritten-underline">1</span>
							<span class="text-gray-700">OF</span>
							<span class="handwritten-underline">1</span>
						</a>
					</div>
				</div>
			</div>
		</div>
	</div>
{:else}
	<!-- Mobile/Tablet Hamburger Button -->
	<button
		on:click={toggleSideMenu}
		class="fixed top-4 left-4 z-50 p-2 bg-white rounded-lg shadow-lg hover:shadow-xl transition-shadow"
		aria-label="Toggle menu"
	>
		<svg class="w-6 h-6 text-gray-700" fill="none" stroke="currentColor" viewBox="0 0 24 24">
			{#if sideMenuOpen}
				<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
			{:else}
				<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16" />
			{/if}
		</svg>
	</button>

	<!-- Side Menu Overlay -->
	{#if sideMenuOpen}
		<div 
			class="fixed inset-0 bg-black bg-opacity-50 z-40"
			on:click={toggleSideMenu}
			on:keydown={() => {}}
			role="button"
			tabindex="0"
		/>
	{/if}

	<!-- Side Menu -->
	<div class="fixed top-0 left-0 h-full w-64 bg-gray-900 z-50 transform transition-transform duration-300 {sideMenuOpen ? 'translate-x-0' : '-translate-x-full'}">
		<div class="flex flex-col h-full">
			<!-- Logo Section -->
			<div class="p-6 border-b border-gray-800">
				<div class="w-32 mx-auto">
					<img 
						src="/giants-head-logo.png" 
						alt="Giants Head Machinery and Design Logo" 
						class="w-full h-full object-contain filter brightness-0 invert"
					/>
				</div>
			</div>

			<!-- Menu Items -->
			<nav class="flex-1 py-6">
				{#each menuItems as item}
					<a
						href={item.href}
						on:click|preventDefault={() => scrollToSection(item.href.substring(1))}
						class="flex items-center px-6 py-4 text-gray-300 hover:bg-gray-800 hover:text-white transition-colors group"
					>
						{#if item.icon === 'user'}
							<svg class="w-5 h-5 mr-4 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M16 7a4 4 0 11-8 0 4 4 0 018 0zM12 14a7 7 0 00-7 7h14a7 7 0 00-7-7z" />
							</svg>
						{:else if item.icon === 'tools'}
							<svg class="w-5 h-5 mr-4 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.065 2.572c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.572 1.065c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.065-2.572c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.996.608 2.296.07 2.572-1.065z" />
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z" />
							</svg>
						{:else if item.icon === 'mail'}
							<svg class="w-5 h-5 mr-4 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 8l7.89 5.26a2 2 0 002.22 0L21 8M5 19h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v10a2 2 0 002 2z" />
							</svg>
						{/if}
						<span class="flex-1 body-font text-sm uppercase tracking-wider">{item.label}</span>
					</a>
				{/each}
			</nav>

			<!-- Profile Section -->
			<div class="p-6 border-t border-gray-800">
				<div class="flex items-center">
					<div class="w-12 h-12 rounded-full overflow-hidden bg-white">
						<img 
							src="/profile-sketch.png" 
							alt="Allen Flett" 
							class="w-full h-full object-cover"
						/>
					</div>
					<div class="ml-3 flex-1">
						<h4 class="text-white font-medium text-sm handwriting">Allen Flett</h4>
						<p class="text-gray-400 text-xs body-font">AScT</p>
					</div>
				</div>
			</div>
		</div>
	</div>
{/if}

<!-- Graph paper sheet below header -->
<div class="graph-paper-sheet {windowWidth <= 1300 ? 'pt-16' : ''}">
	<!-- Content area on graph paper -->
	<main class="body-font">
		<slot />
	</main>
</div>

<style>
	/* Additional styles for side menu */
	:global(body.menu-open) {
		overflow: hidden;
	}
</style>