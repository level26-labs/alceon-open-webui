<script lang="ts">
	import { getContext, createEventDispatcher, onMount } from 'svelte';
	import { fade, fly, slide } from 'svelte/transition';
	import { models as _models, user } from '$lib/stores';

	const dispatch = createEventDispatcher();
	const i18n = getContext('i18n');

	type CapabilityType = 'Prompt' | 'Form' | 'Link' | 'App' | 'Agent' | 'Model' | 'Workflow';

	interface WorkflowStage {
		id: string;
		name: string;
		icon: string;
		color: string;
		prompts: Array<{
			title: string;
			prompt: string;
			features?: PromptFeatures;
			autoSubmit?: boolean;
		}>;
	}

	interface PromptFeatures {
		webSearch?: boolean;
		imageGeneration?: boolean;
		codeInterpreter?: boolean;
	}

	interface FileUploadConfig {
		enabled?: boolean;
		multiple?: boolean;
		accept?: string;  // e.g., ".pdf,.docx,image/*"
		label?: string;   // e.g., "Upload documents"
	}

	interface Capability {
		id: string;
		title: string;
		subtitle: string;
		description: string;
		icon: string;
		color: string;
		capabilityType: CapabilityType;
		action: {
			type: 'model' | 'prompt' | 'route' | 'url' | 'workflow';
			modelId?: string;
			prompt?: string;
			route?: string;
			url?: string;
		};
		features?: PromptFeatures;
		autoSubmit?: boolean;
		fileUpload?: FileUploadConfig;
		workflow?: {
			stages: WorkflowStage[];
		};
		tags?: string[];
		enabled?: boolean;
		// Visibility: 'all' for everyone, or array of group IDs for restricted access
		visibility?: 'all' | string[];
		// New fields for user-friendly features
		examples?: string[];  // Example prompts/uses shown in help tooltip
		difficulty?: 'basic' | 'advanced';  // For basic/advanced mode filtering
		helpText?: string;  // Additional help text for the tooltip
	}

	interface Category {
		id: string;
		label: string;
		default?: boolean;  // If true, this category is selected by default
	}

	interface FeaturedTile {
		id: string;  // Unique ID to track dismissals (change ID to show to all users again)
		title: string;
		subtitle: string;
		description: string;
		icon: string;
		color: string;
		action?: {
			type: 'prompt' | 'route' | 'url';
			prompt?: string;
			route?: string;
			url?: string;
			label?: string;
		};
		enabled: boolean;
	}

	// Configuration loaded from JSON or passed as prop
	let capabilities: Capability[] = [];
	let categories: Category[] = [];
	let featuredTile: FeaturedTile | null = null;
	let configLoaded = false;
	let configError: string | null = null;

	// Config can be passed directly as a prop (inline) OR loaded from URL
	export let config: { categories: Category[]; capabilities: Capability[]; featuredTile?: FeaturedTile } | null = null;
	export let configUrl: string = '';  // Optional URL to load config from
	export let onSelect: (prompt: string, modelId?: string, features?: PromptFeatures, autoSubmit?: boolean, files?: File[]) => void = () => {};
	export let onNavigate: (route: string) => void = () => {};
	// User's group IDs from Open WebUI - pass this in from parent component
	export let userGroups: string[] = [];

	// User preference states (persisted to localStorage)
	let starredCapabilities: string[] = [];
	let dismissedFeaturedTiles: string[] = [];
	
	// Responsive layout - track window height
	let windowHeight = 800;
	$: visibleRows = windowHeight < 625 ? 1 : windowHeight < 800 ? 2 : 3;
	$: showFeaturedOnHeight = windowHeight >= 800;
	
	// Help tooltip state
	let activeHelpTooltip: string | null = null;
	let tooltipContent: string = '';
	let tooltipPosition = { x: 0, y: 0 };
	
	// Show tooltip on hover
	function showTooltip(content: string, event: MouseEvent) {
		const rect = (event.currentTarget as HTMLElement).getBoundingClientRect();
		tooltipContent = content;
		tooltipPosition = {
			x: rect.left + rect.width / 2,
			y: rect.top - 8
		};
		activeHelpTooltip = 'active';
	}
	
	function hideTooltip() {
		activeHelpTooltip = null;
	}
	
	// LocalStorage keys
	const STORAGE_KEY_STARRED = 'capabilities-hub-starred';
	const STORAGE_KEY_DISMISSED_FEATURED = 'capabilities-hub-dismissed-featured';
	
	// Load user preferences from localStorage
	function loadUserPreferences() {
		try {
			const savedStarred = localStorage.getItem(STORAGE_KEY_STARRED);
			if (savedStarred) {
				starredCapabilities = JSON.parse(savedStarred);
			}
			const savedDismissed = localStorage.getItem(STORAGE_KEY_DISMISSED_FEATURED);
			if (savedDismissed) {
				dismissedFeaturedTiles = JSON.parse(savedDismissed);
			}
		} catch (e) {
			console.warn('Failed to load user preferences:', e);
		}
	}
	
	// Dismiss featured tile
	function dismissFeaturedTile() {
		if (featuredTile) {
			dismissedFeaturedTiles = [...dismissedFeaturedTiles, featuredTile.id];
			try {
				localStorage.setItem(STORAGE_KEY_DISMISSED_FEATURED, JSON.stringify(dismissedFeaturedTiles));
			} catch (e) {
				console.warn('Failed to save dismissed featured tiles:', e);
			}
		}
	}
	
	// Check if featured tile should be shown
	$: showFeaturedTile = featuredTile?.enabled && 
		!dismissedFeaturedTiles.includes(featuredTile?.id || '') && 
		selectedCategory === 'all' &&
		showFeaturedOnHeight;
	
	
	// Toggle starred status for a capability
	function toggleStarred(capabilityId: string, event: Event) {
		event.stopPropagation();
		if (starredCapabilities.includes(capabilityId)) {
			starredCapabilities = starredCapabilities.filter(id => id !== capabilityId);
		} else {
			starredCapabilities = [...starredCapabilities, capabilityId];
		}
		try {
			localStorage.setItem(STORAGE_KEY_STARRED, JSON.stringify(starredCapabilities));
		} catch (e) {
			console.warn('Failed to save starred capabilities:', e);
		}
	}
	
	// Check if a capability is starred
	function isStarred(capabilityId: string): boolean {
		return starredCapabilities.includes(capabilityId);
	}
	
	// Toggle help tooltip
	function toggleHelpTooltip(capabilityId: string, event: Event) {
		event.stopPropagation();
		activeHelpTooltip = activeHelpTooltip === capabilityId ? null : capabilityId;
	}
	
	// Close help tooltip when clicking elsewhere
	function closeHelpTooltip() {
		activeHelpTooltip = null;
	}

	// Load configuration from JSON file or use inline config
	async function loadConfig() {
		try {
			// If config is passed as prop, use it directly
			if (config) {
				capabilities = config.capabilities || [];
				categories = config.categories || [];
				featuredTile = config.featuredTile || null;
				configLoaded = true;
				configError = null;
				setTimeout(updateScrollArrows, 100);
				return;
			}

			// If configUrl is provided, fetch from URL
			if (configUrl) {
				const response = await fetch(configUrl);
				if (!response.ok) {
					throw new Error(`Failed to load config: ${response.status}`);
				}
				const loadedConfig = await response.json();
				capabilities = loadedConfig.capabilities || [];
				categories = loadedConfig.categories || [];
				featuredTile = loadedConfig.featuredTile || null;
				configLoaded = true;
				configError = null;
				setTimeout(updateScrollArrows, 100);
				return;
			}

			// No config provided - use empty defaults
			capabilities = [];
			categories = [{ id: 'all', label: 'All' }];
			featuredTile = null;
			configLoaded = true;
			configError = 'No configuration provided. Pass config prop or configUrl.';
		} catch (error) {
			console.error('Error loading capabilities config:', error);
			configError = error instanceof Error ? error.message : 'Unknown error';
			capabilities = [];
			categories = [{ id: 'all', label: 'All' }];
			featuredTile = null;
			configLoaded = true;
		}
	}

	// React to config prop changes
	$: if (config) {
		capabilities = config.capabilities || [];
		categories = config.categories || [];
		featuredTile = config.featuredTile || null;
		configLoaded = true;
		configError = null;
	}

	// Set the initial category based on starred items and config default
	function setInitialCategory() {
		if (starredCapabilities.length > 0) {
			selectedCategory = 'starred';
		} else {
			const defaultCat = categories.find(c => c.default);
			selectedCategory = defaultCat?.id || 'all';
		}
	}

	onMount(async () => {
		loadUserPreferences();  // Load starred items first
		await loadConfig();     // Then load config
		setInitialCategory();   // Set category based on starred items
	});

	// Helper function to check if user can see a capability
	function canUserSeeCapability(capability: Capability): boolean {
		// If no visibility set or set to 'all', everyone can see it
		if (!capability.visibility || capability.visibility === 'all') {
			return true;
		}
		// If visibility is an array of group IDs, check if user is in any of them
		if (Array.isArray(capability.visibility)) {
			return capability.visibility.some(groupId => userGroups.includes(groupId));
		}
		return false;
	}

	// Get default category from config or fall back to 'all'
	$: defaultCategory = categories.find(c => c.default)?.id || 'all';
	let selectedCategory = 'all';
	let previousCategory = 'all';  // Track category before expanded view
	let searchQuery = '';
	let scrollContainer: HTMLDivElement;
	let showLeftArrow = false;
	let showRightArrow = false;
	
	// Expanded view scroll state
	let expandedScrollContainer: HTMLDivElement;
	let showExpandedScrollIndicator = true;

	function updateExpandedScrollIndicator() {
		if (!expandedScrollContainer) return;
		const { scrollTop, scrollHeight, clientHeight } = expandedScrollContainer;
		// Hide indicator when near bottom (within 50px)
		showExpandedScrollIndicator = scrollTop < scrollHeight - clientHeight - 50;
	}
	
	// Expanded view state
	let showExpandedView = false;
	
	// Reset scroll indicator when opening expanded view
	function openExpandedView() {
		previousCategory = selectedCategory;
		selectedCategory = 'all';
		showExpandedView = true;
		showExpandedScrollIndicator = true;
		// Check after render if scrolling is even needed
		setTimeout(() => {
			if (expandedScrollContainer) {
				const { scrollHeight, clientHeight } = expandedScrollContainer;
				showExpandedScrollIndicator = scrollHeight > clientHeight;
			}
		}, 100);
	}
	
	// Close expanded view (restores previous category)
	function closeExpandedView() {
		selectedCategory = previousCategory;
		showExpandedView = false;
	}

	let showInputModal = false;
	let currentCapability: Capability | null = null;
	let currentFeatures: PromptFeatures | null = null;
	let currentAutoSubmit: boolean = true;
	let inputVariables: Array<{
		name: string;
		type: string;
		placeholder: string;
		required: boolean;
		options: string[];
		default: any;
		value: any;
	}> = [];

	// File upload state
	let uploadedFiles: File[] = [];
	let fileInputElement: HTMLInputElement | null = null;

	let showWorkflowModal = false;
	let currentWorkflow: Capability | null = null;
	let selectedStageId: string | null = null;
	let selectedStagePrompt: { stage: WorkflowStage; prompt: { title: string; prompt: string; features?: PromptFeatures; autoSubmit?: boolean } } | null = null;

	$: enabledCapabilities = capabilities.filter((c) => c.enabled !== false && canUserSeeCapability(c));
	
	// Get starred capabilities for the starred section
	$: starredItems = enabledCapabilities.filter((c) => starredCapabilities.includes(c.id));
	
	// Filter by category and search
	$: filteredCapabilities = enabledCapabilities.filter((c) => {
		const matchesCategory = selectedCategory === 'all' || 
			selectedCategory === 'starred' || 
			c.tags?.includes(selectedCategory);
		const matchesSearch = !searchQuery || c.title.toLowerCase().includes(searchQuery.toLowerCase());
		return matchesCategory && matchesSearch;
	});
	
	// When showing starred category, only show starred items
	$: displayCapabilities = selectedCategory === 'starred' 
		? starredItems 
		: filteredCapabilities;

	// Calculate how many tiles fit in one row based on container width
	let containerWidth = 0;
	let tilesPerRow = 4;
	
	$: {
		// Estimate tiles per row: tile width ~280px + gap ~16px = ~296px per tile
		tilesPerRow = Math.max(1, Math.floor((containerWidth || 1000) / 296));
	}

	// Split into three rows: fill first row completely, then second, then third
	$: row1 = displayCapabilities.slice(0, tilesPerRow);
	$: row2 = displayCapabilities.slice(tilesPerRow, tilesPerRow * 2);
	$: row3 = displayCapabilities.slice(tilesPerRow * 2, tilesPerRow * 3);
	$: remainingTiles = displayCapabilities.slice(tilesPerRow * 3);
	
	// Distribute remaining tiles across all three rows for horizontal scroll
	$: row1WithRemaining = [...row1, ...remainingTiles.filter((_, i) => i % 3 === 0)];
	$: row2WithRemaining = [...row2, ...remainingTiles.filter((_, i) => i % 3 === 1)];
	$: row3WithRemaining = [...row3, ...remainingTiles.filter((_, i) => i % 3 === 2)];

	// Filter categories to only show those with matching capabilities
	// Add starred category if there are starred items
	$: visibleCategories = [
		...categories.filter(cat => {
			if (cat.id === 'all') return true;
			return enabledCapabilities.some(c => c.tags?.includes(cat.id));
		}),
		// Add starred category if user has starred items
		...(starredCapabilities.length > 0 ? [{ id: 'starred', label: '⭐ Starred' }] : [])
	];

	$: selectedStage = currentWorkflow?.workflow?.stages.find(s => s.id === selectedStageId) || null;

	$: if (filteredCapabilities && scrollContainer) {
		setTimeout(updateScrollArrows, 100);
	}

	onMount(() => {
		setTimeout(updateScrollArrows, 200);
	});

	const systemVariables = ['CLIPBOARD', 'CURRENT_DATE', 'CURRENT_DATETIME', 'CURRENT_TIME', 'CURRENT_TIMEZONE', 'CURRENT_WEEKDAY', 'USER_NAME', 'USER_LANGUAGE', 'USER_LOCATION'];

	function parseInputVariables(prompt: string) {
		const regex = /\{\{([^}]+)\}\}/g;
		const variables: typeof inputVariables = [];
		let match;

		while ((match = regex.exec(prompt)) !== null) {
			const content = match[1].trim();
			if (systemVariables.includes(content.toUpperCase())) continue;

			const parts = content.split('|').map(p => p.trim());
			const name = parts[0];
			if (systemVariables.includes(name.toUpperCase())) continue;

			let type = 'text', placeholder = '', required = false, options: string[] = [], defaultValue: any = '';

			if (parts[1]) {
				const typeAndProps = parts[1].split(':');
				type = typeAndProps[0] || 'text';
				for (let i = 1; i < typeAndProps.length; i++) {
					const prop = typeAndProps[i];
					if (prop === 'required') required = true;
					else if (prop.startsWith('placeholder=')) placeholder = prop.replace('placeholder=', '').replace(/^["']|["']$/g, '');
					else if (prop.startsWith('default=')) {
						const val = prop.replace('default=', '').replace(/^["']|["']$/g, '');
						defaultValue = type === 'checkbox' ? val === 'true' : val;
					}
					else if (prop.startsWith('options=')) {
						try { options = JSON.parse(prop.replace('options=', '')); } catch { options = []; }
					}
				}
			}

			if (!variables.find(v => v.name === name)) {
				variables.push({ name, type, placeholder: placeholder || name.replace(/_/g, ' '), required, options, default: defaultValue, value: type === 'checkbox' ? (defaultValue || false) : (defaultValue || '') });
			}
		}
		return variables;
	}

	async function replaceSystemVariables(prompt: string): Promise<string> {
		let result = prompt;
		const now = new Date();
		result = result.replace(/\{\{CURRENT_DATE\}\}/gi, now.toLocaleDateString());
		result = result.replace(/\{\{CURRENT_DATETIME\}\}/gi, now.toLocaleString());
		result = result.replace(/\{\{CURRENT_TIME\}\}/gi, now.toLocaleTimeString());
		result = result.replace(/\{\{CURRENT_TIMEZONE\}\}/gi, Intl.DateTimeFormat().resolvedOptions().timeZone);
		result = result.replace(/\{\{CURRENT_WEEKDAY\}\}/gi, now.toLocaleDateString('en-US', { weekday: 'long' }));
		result = result.replace(/\{\{USER_NAME\}\}/gi, $user?.name || 'User');
		result = result.replace(/\{\{USER_LANGUAGE\}\}/gi, navigator.language || 'en');
		if (result.includes('{{CLIPBOARD}}') || result.includes('{{clipboard}}')) {
			try {
				const clipboardText = await navigator.clipboard.readText();
				result = result.replace(/\{\{CLIPBOARD\}\}/gi, clipboardText);
			} catch { result = result.replace(/\{\{CLIPBOARD\}\}/gi, '[Clipboard access denied]'); }
		}
		return result;
	}

	function replaceInputVariables(prompt: string, variables: typeof inputVariables): string {
		let result = prompt;
		for (const v of variables) {
			const patterns = [new RegExp(`\\{\\{${v.name}\\s*\\|[^}]+\\}\\}`, 'g'), new RegExp(`\\{\\{${v.name}\\}\\}`, 'g')];
			const value = v.type === 'checkbox' ? (v.value ? 'Yes' : 'No') : String(v.value || '');
			for (const p of patterns) result = result.replace(p, value);
		}
		return result;
	}

	// Handle file selection
	function handleFileSelect(event: Event) {
		const input = event.target as HTMLInputElement;
		if (input.files) {
			uploadedFiles = Array.from(input.files);
		}
	}

	// Remove a file from the upload list
	function removeFile(index: number) {
		uploadedFiles = uploadedFiles.filter((_, i) => i !== index);
		// Reset the file input
		if (fileInputElement) {
			fileInputElement.value = '';
		}
	}

	// Format file size for display
	function formatFileSize(bytes: number): string {
		if (bytes < 1024) return bytes + ' B';
		if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB';
		return (bytes / (1024 * 1024)).toFixed(1) + ' MB';
	}

	async function handleCapabilityClick(capability: Capability) {
		if (capability.action.type === 'workflow' && capability.workflow) {
			currentWorkflow = capability;
			selectedStageId = capability.workflow.stages[0]?.id || null;
			showWorkflowModal = true;
			return;
		}

		if (capability.action.type === 'model') {
			onSelect('', capability.action.modelId, capability.features, false);
		} else if (capability.action.type === 'prompt') {
			const prompt = capability.action.prompt ?? '';
			const customVars = parseInputVariables(prompt);
			if (customVars.length > 0 || capability.fileUpload?.enabled) {
				currentCapability = capability;
				currentFeatures = capability.features || null;
				currentAutoSubmit = capability.autoSubmit ?? true;
				inputVariables = customVars;
				uploadedFiles = [];  // Reset files
				showInputModal = true;
			} else {
				const processedPrompt = await replaceSystemVariables(prompt);
				onSelect(processedPrompt, undefined, capability.features, capability.autoSubmit ?? false);
			}
		} else if (capability.action.type === 'route') {
			if (capability.action.route) onNavigate(capability.action.route);
		} else if (capability.action.type === 'url') {
			if (capability.action.url) window.open(capability.action.url, '_blank');
		}
		dispatch('select', capability);
	}

	async function handleModalSubmit() {
		if (!currentCapability) return;
		const missingRequired = inputVariables.filter(v => v.required && !v.value && v.value !== false);
		if (missingRequired.length > 0) return;

		let prompt = currentCapability.action.prompt ?? '';
		prompt = await replaceSystemVariables(prompt);
		prompt = replaceInputVariables(prompt, inputVariables);
		onSelect(prompt, undefined, currentFeatures || undefined, currentAutoSubmit, uploadedFiles.length > 0 ? uploadedFiles : undefined);
		closeModal();
	}

	function closeModal() {
		showInputModal = false;
		currentCapability = null;
		currentFeatures = null;
		currentAutoSubmit = true;
		inputVariables = [];
		uploadedFiles = [];
	}

	function closeWorkflowModal() {
		showWorkflowModal = false;
		currentWorkflow = null;
		selectedStageId = null;
		selectedStagePrompt = null;
		currentFeatures = null;
		currentAutoSubmit = true;
		inputVariables = [];
	}

	async function handleWorkflowPromptClick(stage: WorkflowStage, promptItem: { title: string; prompt: string; features?: PromptFeatures; autoSubmit?: boolean }) {
		const customVars = parseInputVariables(promptItem.prompt);
		if (customVars.length > 0) {
			selectedStagePrompt = { stage, prompt: promptItem };
			currentFeatures = promptItem.features || null;
			currentAutoSubmit = promptItem.autoSubmit ?? true;
			inputVariables = customVars;
		} else {
			const processedPrompt = await replaceSystemVariables(promptItem.prompt);
			onSelect(processedPrompt, undefined, promptItem.features, promptItem.autoSubmit ?? true);
			closeWorkflowModal();
		}
	}

	async function handleWorkflowFormSubmit() {
		if (!selectedStagePrompt) return;
		const missingRequired = inputVariables.filter(v => v.required && !v.value && v.value !== false);
		if (missingRequired.length > 0) return;

		let prompt = selectedStagePrompt.prompt.prompt;
		prompt = await replaceSystemVariables(prompt);
		prompt = replaceInputVariables(prompt, inputVariables);
		onSelect(prompt, undefined, currentFeatures || undefined, currentAutoSubmit);
		closeWorkflowModal();
	}

	function formatLabel(name: string): string {
		return name.replace(/_/g, ' ').replace(/\b\w/g, l => l.toUpperCase());
	}

	function updateScrollArrows() {
		if (!scrollContainer) return;
		const { scrollLeft, scrollWidth, clientWidth } = scrollContainer;
		showLeftArrow = scrollLeft > 5;
		showRightArrow = scrollLeft < scrollWidth - clientWidth - 5;
	}

	function scrollLeftFn() { 
		scrollContainer?.scrollBy({ left: -280, behavior: 'smooth' }); 
		setTimeout(updateScrollArrows, 350);
	}
	function scrollRightFn() { 
		scrollContainer?.scrollBy({ left: 280, behavior: 'smooth' }); 
		setTimeout(updateScrollArrows, 350);
	}

	function handleWheel(event: WheelEvent) {
		if (!scrollContainer) return;
		if (scrollContainer.scrollWidth > scrollContainer.clientWidth) {
			event.preventDefault();
			// Increased sensitivity: multiply by 3 for faster scrolling
			scrollContainer.scrollLeft += event.deltaY * 3;
			updateScrollArrows();
		}
	}

	function getBadgeColor(type: CapabilityType): string {
		// Alceon brand colors for badges
		const colors: Record<CapabilityType, string> = {
			'Prompt': 'bg-[#2b6cb0]/10 text-[#2b6cb0] dark:bg-[#63b3ed]/20 dark:text-[#63b3ed]',
			'Form': 'bg-[#c4a35a]/15 text-[#8b7355] dark:bg-[#c4a35a]/20 dark:text-[#d4b896]',
			'Link': 'bg-[#1a365d]/10 text-[#1a365d] dark:bg-[#1a365d]/30 dark:text-[#63b3ed]',
			'App': 'bg-[#63b3ed]/15 text-[#2b6cb0] dark:bg-[#63b3ed]/20 dark:text-[#63b3ed]',
			'Agent': 'bg-[#c4a35a]/15 text-[#8b7355] dark:bg-[#c4a35a]/20 dark:text-[#d4b896]',
			'Model': 'bg-[#1a365d]/15 text-[#1a365d] dark:bg-[#1a365d]/30 dark:text-[#63b3ed]',
			'Workflow': 'bg-[#1a365d]/10 text-[#1a365d] dark:bg-[#2b6cb0]/20 dark:text-[#63b3ed]'
		};
		return colors[type] || colors['Prompt'];
	}

	function getFeatureIcons(features?: PromptFeatures): Array<{icon: string, label: string}> {
		if (!features) return [];
		const icons = [];
		if (features.webSearch) icons.push({ icon: '🌐', label: 'Web Search' });
		if (features.imageGeneration) icons.push({ icon: '🎨', label: 'Image Generation' });
		if (features.codeInterpreter) icons.push({ icon: '▶️', label: 'Code Interpreter' });
		return icons;
	}

	function renderTile(capability: Capability, idx: number) {
		return capability;
	}
</script>

<svelte:window bind:innerHeight={windowHeight} />

{#if !configLoaded}
	<!-- Loading state -->
	<div class="capabilities-hub w-full" in:fade={{ duration: 300 }}>
		<div class="mb-3 sm:mb-4">
			<h1 class="text-lg sm:text-xl font-semibold text-gray-800 dark:text-gray-100">What would you like to do?</h1>
		</div>
		<div class="flex items-center justify-center h-[290px] sm:h-[348px]">
			<div class="text-gray-400 dark:text-gray-500">Loading capabilities...</div>
		</div>
	</div>
{:else if configError}
	<!-- Error state -->
	<div class="capabilities-hub w-full" in:fade={{ duration: 300 }}>
		<div class="mb-3 sm:mb-4">
			<h1 class="text-lg sm:text-xl font-semibold text-gray-800 dark:text-gray-100">What would you like to do?</h1>
		</div>
		<div class="flex flex-col items-center justify-center h-[290px] sm:h-[348px] text-center">
			<div class="text-red-500 dark:text-red-400 mb-2">Failed to load capabilities</div>
			<div class="text-sm text-gray-500 dark:text-gray-400">{configError}</div>
			<button 
				class="mt-3 px-4 py-2 text-sm bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition-colors"
				on:click={loadConfig}
			>
				Retry
			</button>
		</div>
	</div>
{:else}
<div class="capabilities-hub w-full" in:fade={{ duration: 300 }}>
	<div class="mb-3 sm:mb-4">
		<h1 class="text-lg sm:text-xl font-semibold text-gray-800 dark:text-gray-100">What would you like to do?</h1>
	</div>

	<!-- Categories + Search -->
	<div class="flex items-center gap-2 mb-3 sm:mb-4 flex-wrap">
		<div class="flex items-center gap-1.5 sm:gap-2 flex-wrap">
			{#each visibleCategories as category}
				<button
					class="px-2 sm:px-3 py-1 rounded-full text-xs sm:text-sm font-medium transition-all duration-200
						   {selectedCategory === category.id ? 'bg-gray-900 dark:bg-white text-white dark:text-gray-900' : 'bg-gray-100 dark:bg-gray-800 text-gray-600 dark:text-gray-400 hover:bg-gray-200 dark:hover:bg-gray-700'}"
					on:click={() => (selectedCategory = category.id)}
				>
					{category.label}
				</button>
			{/each}
		</div>
		<div class="flex items-center gap-2">
			<div class="relative hidden sm:block">
				<input
					type="text"
					placeholder="Search..."
					bind:value={searchQuery}
					class="w-40 px-3 py-1 pl-8 rounded-full text-sm border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800 text-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500/50"
				/>
				<svg class="absolute left-2.5 top-1/2 -translate-y-1/2 w-3.5 h-3.5 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" />
				</svg>
			</div>
			<!-- Expand button - desktop only -->
			<button
				class="hidden sm:flex p-1.5 rounded-lg border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800 text-gray-500 dark:text-gray-400 hover:bg-gray-50 dark:hover:bg-gray-700 hover:text-gray-700 dark:hover:text-gray-200 transition-colors items-center justify-center"
				on:click={openExpandedView}
				title="Expand view"
			>
				<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 8V4m0 0h4M4 4l5 5m11-1V4m0 0h-4m4 0l-5 5M4 16v4m0 0h4m-4 0l5-5m11 5l-5-5m5 5v-4m0 4h-4" />
				</svg>
			</button>
		</div>
	</div>

	<div class="relative overflow-visible" style="height: {visibleRows === 1 ? '108px' : visibleRows === 2 ? '224px' : '340px'};">
		{#if showLeftArrow}
			<button 
				class="absolute -left-2 sm:-left-3 top-1/2 -translate-y-1/2 z-20 w-7 h-7 sm:w-8 sm:h-8 rounded-full bg-white dark:bg-gray-800 shadow-lg border border-gray-200 dark:border-gray-700 flex items-center justify-center hover:bg-gray-50 dark:hover:bg-gray-700 transition-all"
				style="margin-top: -22px;"
				on:click={scrollLeftFn}
				in:fade={{ duration: 150 }}
			>
				<svg class="w-3.5 h-3.5 sm:w-4 sm:h-4 text-gray-600 dark:text-gray-300" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" /></svg>
			</button>
		{/if}

		{#if showRightArrow}
			<button 
				class="absolute -right-2 sm:-right-3 top-1/2 -translate-y-1/2 z-20 w-7 h-7 sm:w-8 sm:h-8 rounded-full bg-white dark:bg-gray-800 shadow-lg border border-gray-200 dark:border-gray-700 flex items-center justify-center hover:bg-gray-50 dark:hover:bg-gray-700 transition-all"
				style="margin-top: 0px;"
				on:click={scrollRightFn}
				in:fade={{ duration: 150 }}
			>
				<svg class="w-3.5 h-3.5 sm:w-4 sm:h-4 text-gray-600 dark:text-gray-300" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" /></svg>
			</button>
		{/if}

		{#if showRightArrow}
			<div class="absolute right-0 top-0 bottom-0 w-12 sm:w-16 bg-gradient-to-l from-white dark:from-gray-900 to-transparent pointer-events-none z-10" />
		{/if}

		{#if showLeftArrow}
			<div class="absolute left-0 top-0 bottom-0 w-12 sm:w-16 bg-gradient-to-r from-white dark:from-gray-900 to-transparent pointer-events-none z-10" />
		{/if}

		<!-- Fixed height three-row grid layout -->
		<div 
			bind:this={scrollContainer} 
			bind:clientWidth={containerWidth}
			on:scroll={updateScrollArrows} 
			on:wheel={handleWheel}
			class="overflow-x-auto overflow-y-visible px-1 scroll-smooth scrollbar-hide h-full"
		>
			<div class="flex gap-3 sm:gap-4 h-full pt-1">
				<!-- Featured Tile - spans all 3 rows when on "All" category -->
				{#if showFeaturedTile && featuredTile}
					<div 
						class="featured-tile flex-shrink-0 w-[280px] rounded-xl p-4 sm:p-5 relative
							   bg-gradient-to-br {featuredTile.color} text-white
							   border border-white/20"
						style="height: calc(100% - 4px);"
						in:fly={{ x: -20, duration: 300 }}
					>
						<!-- Dismiss button -->
						<button
							class="absolute top-2 right-2 w-6 h-6 rounded-full bg-white/20 hover:bg-white/30 flex items-center justify-center transition-colors"
							on:click|stopPropagation={dismissFeaturedTile}
							title="Dismiss"
						>
							<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" /></svg>
						</button>
						
						<!-- Content -->
						<div class="flex flex-col h-full">
							<div class="w-12 h-12 sm:w-14 sm:h-14 rounded-xl flex items-center justify-center bg-white/20 mb-4">
								<span class="text-2xl sm:text-3xl">{featuredTile.icon}</span>
							</div>
							
							<h3 class="font-bold text-lg sm:text-xl mb-1">{featuredTile.title}</h3>
							<p class="text-sm text-white/80 mb-3">{featuredTile.subtitle}</p>
							
							<p class="text-sm text-white/70 flex-1">{featuredTile.description}</p>
							
							{#if featuredTile.action}
								<button
									class="mt-4 px-4 py-2 bg-white/20 hover:bg-white/30 rounded-lg text-sm font-medium transition-colors"
									on:click|stopPropagation={() => {
										if (featuredTile?.action?.type === 'prompt' && featuredTile.action.prompt) {
											onSelect(featuredTile.action.prompt, undefined, undefined, false);
										} else if (featuredTile?.action?.type === 'route' && featuredTile.action.route) {
											onNavigate(featuredTile.action.route);
										} else if (featuredTile?.action?.type === 'url' && featuredTile.action.url) {
											window.open(featuredTile.action.url, '_blank');
										}
									}}
								>
									{featuredTile.action.label || 'Learn More'} →
								</button>
							{/if}
						</div>
					</div>
				{/if}
				
				<!-- Regular tiles in 3 rows -->
				<div class="flex flex-col gap-3 sm:gap-4 flex-1">
					<!-- Row 1 -->
					<div class="flex gap-3 sm:gap-4">
						{#each row1WithRemaining as capability, idx (capability.id)}
							<!-- svelte-ignore a11y-no-static-element-interactions -->
							<!-- svelte-ignore a11y-click-events-have-key-events -->
							<div
								class="capability-card group relative flex-shrink-0 w-[280px] rounded-xl p-3 sm:p-4 text-left cursor-pointer
									   bg-white dark:bg-gray-800/50 border border-gray-200 dark:border-gray-700/50
									   hover:border-gray-300 dark:hover:border-gray-600 hover:shadow-lg
									   transform hover:-translate-y-0.5 transition-all duration-200"
								on:click={() => handleCapabilityClick(capability)}
								in:fly={{ x: 20, duration: 200, delay: idx * 30 }}
							>
								<!-- Top row: Badge left, icons right -->
								<div class="flex items-center justify-between mb-3">
									<span class="text-[9px] sm:text-[10px] font-medium px-1.5 py-0.5 rounded {getBadgeColor(capability.capabilityType)}">{capability.capabilityType}</span>
									<div class="flex items-center gap-0.5">
										{#if capability.description || capability.helpText}
											<div
												class="w-5 h-5 rounded-full flex items-center justify-center text-gray-400 hover:text-blue-500 hover:bg-blue-50 dark:hover:bg-blue-900/30 transition-colors opacity-0 group-hover:opacity-100 cursor-help"
												on:mouseenter={(e) => showTooltip(capability.description + (capability.helpText ? '\n\n' + capability.helpText : ''), e)}
												on:mouseleave={hideTooltip}
											>
												<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" /></svg>
											</div>
										{/if}
										<button
											class="w-5 h-5 rounded-full flex items-center justify-center transition-colors
												   {isStarred(capability.id) 
												   	? 'text-[#63b3ed]' 
												   	: 'text-gray-300 hover:text-[#63b3ed] opacity-0 group-hover:opacity-100'}"
											on:click|stopPropagation={(e) => toggleStarred(capability.id, e)}
											title={isStarred(capability.id) ? 'Remove from starred' : 'Add to starred'}
										>
											<svg class="w-3.5 h-3.5" fill={isStarred(capability.id) ? 'currentColor' : 'none'} stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11.049 2.927c.3-.921 1.603-.921 1.902 0l1.519 4.674a1 1 0 00.95.69h4.915c.969 0 1.371 1.24.588 1.81l-3.976 2.888a1 1 0 00-.363 1.118l1.518 4.674c.3.922-.755 1.688-1.538 1.118l-3.976-2.888a1 1 0 00-1.176 0l-3.976 2.888c-.783.57-1.838-.197-1.538-1.118l1.518-4.674a1 1 0 00-.363-1.118l-3.976-2.888c-.784-.57-.38-1.81.588-1.81h4.914a1 1 0 00.951-.69l1.519-4.674z" /></svg>
										</button>
								</div>
							</div>

							<div class="flex items-center gap-2.5">
								<div class="w-8 h-8 rounded-lg flex items-center justify-center flex-shrink-0 bg-gradient-to-br {capability.color} shadow-sm group-hover:scale-110 transition-transform duration-200">
									<span class="text-base">{capability.icon}</span>
								</div>
								<div class="min-w-0 flex-1">
									<h3 class="font-semibold text-sm text-gray-800 dark:text-gray-100 truncate leading-tight">{capability.title}</h3>
									<p class="text-xs text-gray-500 dark:text-gray-400 truncate leading-tight mt-0.5">{capability.subtitle}</p>
								</div>
							</div>
						</div>
					{/each}
				</div>

				<!-- Row 2 -->
				{#if visibleRows >= 2}
				<div class="flex gap-3 sm:gap-4">
					{#each row2WithRemaining as capability, idx (capability.id)}
						<!-- svelte-ignore a11y-no-static-element-interactions -->
						<!-- svelte-ignore a11y-click-events-have-key-events -->
						<div
							class="capability-card group relative flex-shrink-0 w-[280px] rounded-xl p-3 sm:p-4 text-left cursor-pointer
								   bg-white dark:bg-gray-800/50 border border-gray-200 dark:border-gray-700/50
								   hover:border-gray-300 dark:hover:border-gray-600 hover:shadow-lg
								   transform hover:-translate-y-0.5 transition-all duration-200"
							on:click={() => handleCapabilityClick(capability)}
							in:fly={{ x: 20, duration: 200, delay: idx * 30 + 50 }}
						>
							<!-- Top row: Badge left, icons right -->
							<div class="flex items-center justify-between mb-3">
								<span class="text-[9px] sm:text-[10px] font-medium px-1.5 py-0.5 rounded {getBadgeColor(capability.capabilityType)}">{capability.capabilityType}</span>
								<div class="flex items-center gap-0.5">
									{#if capability.description || capability.helpText}
										<div
											class="w-5 h-5 rounded-full flex items-center justify-center text-gray-400 hover:text-blue-500 hover:bg-blue-50 dark:hover:bg-blue-900/30 transition-colors opacity-0 group-hover:opacity-100 cursor-help"
											on:mouseenter={(e) => showTooltip(capability.description + (capability.helpText ? '\n\n' + capability.helpText : ''), e)}
											on:mouseleave={hideTooltip}
										>
											<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" /></svg>
										</div>
									{/if}
									<button
										class="w-5 h-5 rounded-full flex items-center justify-center transition-colors
											   {isStarred(capability.id) 
											   	? 'text-[#63b3ed]' 
											   	: 'text-gray-300 hover:text-[#63b3ed] opacity-0 group-hover:opacity-100'}"
										on:click|stopPropagation={(e) => toggleStarred(capability.id, e)}
										title={isStarred(capability.id) ? 'Remove from starred' : 'Add to starred'}
									>
										<svg class="w-3.5 h-3.5" fill={isStarred(capability.id) ? 'currentColor' : 'none'} stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11.049 2.927c.3-.921 1.603-.921 1.902 0l1.519 4.674a1 1 0 00.95.69h4.915c.969 0 1.371 1.24.588 1.81l-3.976 2.888a1 1 0 00-.363 1.118l1.518 4.674c.3.922-.755 1.688-1.538 1.118l-3.976-2.888a1 1 0 00-1.176 0l-3.976 2.888c-.783.57-1.838-.197-1.538-1.118l1.518-4.674a1 1 0 00-.363-1.118l-3.976-2.888c-.784-.57-.38-1.81.588-1.81h4.914a1 1 0 00.951-.69l1.519-4.674z" /></svg>
									</button>
								</div>
							</div>

							<div class="flex items-center gap-2.5">
								<div class="w-8 h-8 rounded-lg flex items-center justify-center flex-shrink-0 bg-gradient-to-br {capability.color} shadow-sm group-hover:scale-110 transition-transform duration-200">
									<span class="text-base">{capability.icon}</span>
								</div>
								<div class="min-w-0 flex-1">
									<h3 class="font-semibold text-sm text-gray-800 dark:text-gray-100 truncate leading-tight">{capability.title}</h3>
									<p class="text-xs text-gray-500 dark:text-gray-400 truncate leading-tight mt-0.5">{capability.subtitle}</p>
								</div>
							</div>
						</div>
					{/each}
				</div>
				{/if}

				<!-- Row 3 -->
				{#if visibleRows >= 3}
				<div class="flex gap-3 sm:gap-4">
					{#each row3WithRemaining as capability, idx (capability.id)}
						<!-- svelte-ignore a11y-no-static-element-interactions -->
						<!-- svelte-ignore a11y-click-events-have-key-events -->
						<div
							class="capability-card group relative flex-shrink-0 w-[280px] rounded-xl p-3 sm:p-4 text-left cursor-pointer
								   bg-white dark:bg-gray-800/50 border border-gray-200 dark:border-gray-700/50
								   hover:border-gray-300 dark:hover:border-gray-600 hover:shadow-lg
								   transform hover:-translate-y-0.5 transition-all duration-200"
							on:click={() => handleCapabilityClick(capability)}
							in:fly={{ x: 20, duration: 200, delay: idx * 30 + 100 }}
						>
							<!-- Top row: Badge left, icons right -->
							<div class="flex items-center justify-between mb-3">
								<span class="text-[9px] sm:text-[10px] font-medium px-1.5 py-0.5 rounded {getBadgeColor(capability.capabilityType)}">{capability.capabilityType}</span>
								<div class="flex items-center gap-0.5">
									{#if capability.description || capability.helpText}
										<div
											class="w-5 h-5 rounded-full flex items-center justify-center text-gray-400 hover:text-blue-500 hover:bg-blue-50 dark:hover:bg-blue-900/30 transition-colors opacity-0 group-hover:opacity-100 cursor-help"
											on:mouseenter={(e) => showTooltip(capability.description + (capability.helpText ? '\n\n' + capability.helpText : ''), e)}
											on:mouseleave={hideTooltip}
										>
											<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" /></svg>
										</div>
									{/if}
									<button
										class="w-5 h-5 rounded-full flex items-center justify-center transition-colors
											   {isStarred(capability.id) 
											   	? 'text-[#63b3ed]' 
											   	: 'text-gray-300 hover:text-[#63b3ed] opacity-0 group-hover:opacity-100'}"
										on:click|stopPropagation={(e) => toggleStarred(capability.id, e)}
										title={isStarred(capability.id) ? 'Remove from starred' : 'Add to starred'}
									>
										<svg class="w-3.5 h-3.5" fill={isStarred(capability.id) ? 'currentColor' : 'none'} stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11.049 2.927c.3-.921 1.603-.921 1.902 0l1.519 4.674a1 1 0 00.95.69h4.915c.969 0 1.371 1.24.588 1.81l-3.976 2.888a1 1 0 00-.363 1.118l1.518 4.674c.3.922-.755 1.688-1.538 1.118l-3.976-2.888a1 1 0 00-1.176 0l-3.976 2.888c-.783.57-1.838-.197-1.538-1.118l1.518-4.674a1 1 0 00-.363-1.118l-3.976-2.888c-.784-.57-.38-1.81.588-1.81h4.914a1 1 0 00.951-.69l1.519-4.674z" /></svg>
									</button>
								</div>
							</div>

							<div class="flex items-center gap-2.5">
								<div class="w-8 h-8 rounded-lg flex items-center justify-center flex-shrink-0 bg-gradient-to-br {capability.color} shadow-sm group-hover:scale-110 transition-transform duration-200">
									<span class="text-base">{capability.icon}</span>
								</div>
								<div class="min-w-0 flex-1">
									<h3 class="font-semibold text-sm text-gray-800 dark:text-gray-100 truncate leading-tight">{capability.title}</h3>
									<p class="text-xs text-gray-500 dark:text-gray-400 truncate leading-tight mt-0.5">{capability.subtitle}</p>
								</div>
							</div>
						</div>
					{/each}
				</div>
				{/if}
			</div>
		</div>
	</div>
</div>

<!-- Help text with channel links -->
<p class="text-xs text-gray-500 dark:text-gray-400 mt-4">
	Help us make Kingfisher better by reporting 
	<a href="/channels/dc28eb0c-d5ed-45e1-bd99-fd1645a877a1" class="text-blue-500 hover:text-blue-600 dark:text-blue-400 dark:hover:text-blue-300 hover:underline">issues</a> 
	and submitting 
	<a href="/channels/42a93a10-16df-4bc4-a5a1-da1c25737a0a" class="text-blue-500 hover:text-blue-600 dark:text-blue-400 dark:hover:text-blue-300 hover:underline">ideas</a> 
	for new prompts, workflows or functionality.
</p>

</div>
{/if}

<!-- Expanded View Modal -->
{#if showExpandedView}
	<div class="absolute inset-0 top-12 bottom-[160px] bg-white dark:bg-gray-900 z-50 flex flex-col overflow-hidden rounded-xl" in:fade={{ duration: 200 }}>
		<!-- Header -->
		<div class="flex items-center justify-between p-4 border-b border-gray-200 dark:border-gray-700">
			<h2 class="text-lg font-semibold text-gray-800 dark:text-gray-100">What would you like to do?</h2>
			<div class="flex items-center gap-3">
				<!-- Search in expanded view -->
				<div class="relative">
					<input
						type="text"
						placeholder="Search..."
						bind:value={searchQuery}
						class="w-48 px-3 py-1.5 pl-8 rounded-lg text-sm border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800 text-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500/50"
					/>
					<svg class="absolute left-2.5 top-1/2 -translate-y-1/2 w-4 h-4 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" />
					</svg>
				</div>
				<!-- Close button -->
				<button
					class="p-2 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800 text-gray-500 dark:text-gray-400 transition-colors"
					on:click={closeExpandedView}
					title="Close"
				>
					<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
					</svg>
				</button>
			</div>
		</div>
		
		<!-- Category tabs -->
		<div class="flex items-center gap-2 p-4 border-b border-gray-200 dark:border-gray-700 flex-wrap">
			{#each visibleCategories as category}
				<button
					class="px-3 py-1.5 rounded-full text-sm font-medium transition-all duration-200
						   {selectedCategory === category.id ? 'bg-gray-900 dark:bg-white text-white dark:text-gray-900' : 'bg-gray-100 dark:bg-gray-800 text-gray-600 dark:text-gray-400 hover:bg-gray-200 dark:hover:bg-gray-700'}"
					on:click={() => (selectedCategory = category.id)}
				>
					{category.label}
				</button>
			{/each}
		</div>
		
		<!-- All tiles grid -->
		<div 
			class="flex-1 overflow-y-auto p-4 relative"
			bind:this={expandedScrollContainer}
			on:scroll={updateExpandedScrollIndicator}
		>
			<div class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 xl:grid-cols-5 gap-4">
				{#each displayCapabilities as capability (capability.id)}
					<!-- svelte-ignore a11y-no-static-element-interactions -->
					<!-- svelte-ignore a11y-click-events-have-key-events -->
					<div
						class="capability-card group relative rounded-xl p-3 sm:p-4 text-left cursor-pointer
							   bg-white dark:bg-gray-800/50 border border-gray-200 dark:border-gray-700/50
							   hover:border-gray-300 dark:hover:border-gray-600 hover:shadow-lg
							   transform hover:-translate-y-0.5 transition-all duration-200"
						on:click={() => { closeExpandedView(); handleCapabilityClick(capability); }}
					>
						<!-- Top row: Badge left, icons right -->
						<div class="flex items-center justify-between mb-3">
							<span class="text-[9px] sm:text-[10px] font-medium px-1.5 py-0.5 rounded {getBadgeColor(capability.capabilityType)}">{capability.capabilityType}</span>
							<div class="flex items-center gap-0.5">
								{#if capability.description || capability.helpText}
									<div
										class="w-5 h-5 rounded-full flex items-center justify-center text-gray-400 hover:text-blue-500 hover:bg-blue-50 dark:hover:bg-blue-900/30 transition-colors opacity-0 group-hover:opacity-100 cursor-help"
										on:mouseenter={(e) => showTooltip(capability.description + (capability.helpText ? '\n\n' + capability.helpText : ''), e)}
										on:mouseleave={hideTooltip}
									>
										<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" /></svg>
									</div>
								{/if}
								<button
									class="w-5 h-5 rounded-full flex items-center justify-center transition-colors
										   {isStarred(capability.id) 
										   	? 'text-[#63b3ed]' 
										   	: 'text-gray-300 hover:text-[#63b3ed] opacity-0 group-hover:opacity-100'}"
									on:click|stopPropagation={(e) => toggleStarred(capability.id, e)}
									title={isStarred(capability.id) ? 'Remove from starred' : 'Add to starred'}
								>
									<svg class="w-3.5 h-3.5" fill={isStarred(capability.id) ? 'currentColor' : 'none'} stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11.049 2.927c.3-.921 1.603-.921 1.902 0l1.519 4.674a1 1 0 00.95.69h4.915c.969 0 1.371 1.24.588 1.81l-3.976 2.888a1 1 0 00-.363 1.118l1.518 4.674c.3.922-.755 1.688-1.538 1.118l-3.976-2.888a1 1 0 00-1.176 0l-3.976 2.888c-.783.57-1.838-.197-1.538-1.118l1.518-4.674a1 1 0 00-.363-1.118l-3.976-2.888c-.784-.57-.38-1.81.588-1.81h4.914a1 1 0 00.951-.69l1.519-4.674z" /></svg>
								</button>
							</div>
						</div>

						<div class="flex items-center gap-2.5">
							<div class="w-8 h-8 rounded-lg flex items-center justify-center flex-shrink-0 bg-gradient-to-br {capability.color} shadow-sm group-hover:scale-110 transition-transform duration-200">
								<span class="text-base">{capability.icon}</span>
							</div>
							<div class="min-w-0 flex-1">
								<h3 class="font-semibold text-sm text-gray-800 dark:text-gray-100 truncate leading-tight">{capability.title}</h3>
								<p class="text-xs text-gray-500 dark:text-gray-400 truncate leading-tight mt-0.5">{capability.subtitle}</p>
							</div>
						</div>
					</div>
				{/each}
			</div>
		</div>
		
		<!-- Scroll indicator -->
		{#if showExpandedScrollIndicator}
			<div 
				class="absolute bottom-0 left-0 right-0 flex justify-center pointer-events-none"
				in:fade={{ duration: 150 }}
				out:fade={{ duration: 150 }}
			>
				<div class="bg-gradient-to-t from-white dark:from-gray-900 via-white/80 dark:via-gray-900/80 to-transparent pt-8 pb-3 px-6 w-full flex justify-center">
					<div class="flex items-center gap-1.5 text-xs text-gray-400 dark:text-gray-500 animate-bounce">
						<span>Scroll for more</span>
						<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 14l-7 7m0 0l-7-7m7 7V3" />
						</svg>
					</div>
				</div>
			</div>
		{/if}
	</div>
{/if}

<!-- Standard Input Modal -->
{#if showInputModal && currentCapability}
	<div class="fixed inset-0 bg-black/50 flex items-center justify-center z-[100] p-3 sm:p-4" on:click|self={closeModal} in:fade={{ duration: 200 }}>
		<div class="bg-white dark:bg-gray-800 rounded-2xl shadow-2xl max-w-lg w-full max-h-[90vh] overflow-hidden" in:fly={{ y: 20, duration: 300 }}>
			<div class="flex items-center gap-3 p-3 sm:p-4 border-b border-gray-200 dark:border-gray-700">
				<div class="w-8 h-8 rounded-lg flex items-center justify-center bg-gradient-to-br {currentCapability.color}"><span class="text-base">{currentCapability.icon}</span></div>
				<div class="flex-1">
					<h2 class="font-semibold text-gray-800 dark:text-gray-100">{currentCapability.title}</h2>
				</div>
				<button class="p-1.5 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-lg" on:click={closeModal}>
					<svg class="w-4 h-4 text-gray-500" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" /></svg>
				</button>
			</div>
			<div class="p-3 sm:p-4 overflow-y-auto max-h-[60vh]">
				<form on:submit|preventDefault={handleModalSubmit} class="space-y-3">
					{#each inputVariables as variable}
						<div class="space-y-1">
							<label class="block text-sm font-medium text-gray-700 dark:text-gray-300">{formatLabel(variable.name)}{#if variable.required}<span class="text-red-500">*</span>{/if}</label>
							{#if variable.type === 'textarea'}
								<textarea bind:value={variable.value} placeholder={variable.placeholder || `e.g., Enter your ${formatLabel(variable.name).toLowerCase()} here...`} required={variable.required} rows="3" class="w-full px-3 py-2 text-sm rounded-lg border border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-700 text-gray-800 dark:text-gray-200 placeholder:text-gray-400 dark:placeholder:text-gray-500 focus:outline-none focus:ring-2 focus:ring-blue-500/50 resize-none" />
							{:else if variable.type === 'select'}
								<select bind:value={variable.value} required={variable.required} class="w-full px-3 py-2 text-sm rounded-lg border border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-700 text-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500/50">
									<option value="">Select...</option>
									{#each variable.options as option}<option value={option}>{option}</option>{/each}
								</select>
							{:else if variable.type === 'checkbox'}
								<label class="flex items-center gap-2 cursor-pointer"><input type="checkbox" bind:checked={variable.value} class="w-4 h-4 rounded" /><span class="text-sm text-gray-600 dark:text-gray-400">{variable.placeholder}</span></label>
							{:else if variable.type === 'date'}
								<input type="date" bind:value={variable.value} required={variable.required} class="w-full px-3 py-2 text-sm rounded-lg border border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-700 text-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500/50" />
							{:else}
								<input type="text" bind:value={variable.value} placeholder={variable.placeholder || `e.g., Enter ${formatLabel(variable.name).toLowerCase()}...`} required={variable.required} class="w-full px-3 py-2 text-sm rounded-lg border border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-700 text-gray-800 dark:text-gray-200 placeholder:text-gray-400 dark:placeholder:text-gray-500 focus:outline-none focus:ring-2 focus:ring-blue-500/50" />
							{/if}
						</div>
					{/each}

					<!-- File Upload Section -->
					{#if currentCapability.fileUpload?.enabled}
						<div class="space-y-2 pt-2 border-t border-gray-100 dark:border-gray-700">
							<label class="block text-sm font-medium text-gray-700 dark:text-gray-300">
								{currentCapability.fileUpload.label || 'Upload Files'}
							</label>
							
							<!-- File input -->
							<div class="flex items-center gap-2">
								<input
									bind:this={fileInputElement}
									type="file"
									accept={currentCapability.fileUpload.accept || '*/*'}
									multiple={currentCapability.fileUpload.multiple ?? true}
									on:change={handleFileSelect}
									class="hidden"
									id="file-upload-input"
								/>
								<label 
									for="file-upload-input"
									class="flex items-center gap-2 px-3 py-2 text-sm rounded-lg border border-dashed border-gray-300 dark:border-gray-600 bg-gray-50 dark:bg-gray-700/50 text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700 hover:border-gray-400 dark:hover:border-gray-500 cursor-pointer transition-colors"
								>
									<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
										<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" />
									</svg>
									<span>Choose files...</span>
								</label>
								{#if uploadedFiles.length > 0}
									<span class="text-xs text-gray-500">{uploadedFiles.length} file{uploadedFiles.length > 1 ? 's' : ''} selected</span>
								{/if}
							</div>

							<!-- File list -->
							{#if uploadedFiles.length > 0}
								<div class="space-y-1.5 mt-2">
									{#each uploadedFiles as file, index}
										<div class="flex items-center justify-between px-3 py-2 bg-gray-50 dark:bg-gray-700/50 rounded-lg">
											<div class="flex items-center gap-2 min-w-0 flex-1">
												<svg class="w-4 h-4 text-gray-400 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
													<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
												</svg>
												<span class="text-sm text-gray-700 dark:text-gray-300 truncate">{file.name}</span>
												<span class="text-xs text-gray-400 flex-shrink-0">({formatFileSize(file.size)})</span>
											</div>
											<button
												type="button"
												class="p-1 hover:bg-gray-200 dark:hover:bg-gray-600 rounded transition-colors"
												on:click={() => removeFile(index)}
												title="Remove file"
											>
												<svg class="w-3.5 h-3.5 text-gray-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
													<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
												</svg>
											</button>
										</div>
									{/each}
								</div>
							{/if}
						</div>
					{/if}
				</form>
			</div>
			<div class="flex justify-end gap-2 p-3 sm:p-4 border-t border-gray-200 dark:border-gray-700">
				<button type="button" class="px-3 py-1.5 text-sm rounded-lg text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700" on:click={closeModal}>Cancel</button>
				<button type="button" class="px-3 py-1.5 text-sm rounded-lg bg-blue-500 text-white hover:bg-blue-600 font-medium" on:click={handleModalSubmit}>
					{currentAutoSubmit ? 'Submit' : 'Load Prompt'}
				</button>
			</div>
		</div>
	</div>
{/if}

<!-- Workflow Modal -->
{#if showWorkflowModal && currentWorkflow?.workflow}
	<div class="fixed inset-0 bg-black/50 flex items-center justify-center z-[100] p-3 sm:p-4" on:click|self={closeWorkflowModal} in:fade={{ duration: 200 }}>
		<div class="bg-white dark:bg-gray-800 rounded-2xl shadow-2xl max-w-3xl w-full max-h-[90vh] overflow-hidden" in:fly={{ y: 20, duration: 300 }}>
			<!-- Header -->
			<div class="flex items-center gap-3 p-3 sm:p-4 border-b border-gray-200 dark:border-gray-700">
				<div class="w-9 h-9 sm:w-10 sm:h-10 rounded-xl flex items-center justify-center bg-gradient-to-br {currentWorkflow.color}"><span class="text-lg sm:text-xl">{currentWorkflow.icon}</span></div>
				<div class="flex-1 min-w-0">
					<h2 class="font-semibold text-base sm:text-lg text-gray-800 dark:text-gray-100">{currentWorkflow.title}</h2>
					<p class="text-xs sm:text-sm text-gray-500 dark:text-gray-400 truncate">{currentWorkflow.description}</p>
				</div>
				<button class="p-1.5 sm:p-2 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-lg" on:click={closeWorkflowModal}>
					<svg class="w-4 h-4 sm:w-5 sm:h-5 text-gray-500" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" /></svg>
				</button>
			</div>

			{#if !selectedStagePrompt}
				<div class="flex flex-col">
					<!-- Prominent Timeline -->
					<div class="px-3 sm:px-6 py-4 sm:py-6 border-b border-gray-100 dark:border-gray-700/50 bg-gradient-to-b from-gray-50 to-white dark:from-gray-800/50 dark:to-gray-800">
						<div class="flex items-center justify-between gap-1 sm:gap-2">
							{#each currentWorkflow.workflow.stages as stage, idx}
								<button
									class="flex-1 flex flex-col items-center gap-1 sm:gap-2 py-2 sm:py-3 px-1 rounded-xl transition-all
										   {selectedStageId === stage.id 
										   	? 'bg-white dark:bg-gray-700 shadow-md ring-2 ring-blue-500/30' 
										   	: 'hover:bg-white/80 dark:hover:bg-gray-700/50 hover:shadow-sm'}"
									on:click={() => selectedStageId = stage.id}
								>
									<div 
										class="w-12 h-12 sm:w-14 sm:h-14 rounded-full bg-gradient-to-br {stage.color} shadow-lg flex items-center justify-center
											   transition-all duration-200 {selectedStageId === stage.id ? 'scale-110 ring-4 ring-white dark:ring-gray-700' : 'hover:scale-105'}"
									>
										<span class="text-xl sm:text-2xl">{stage.icon}</span>
									</div>
									<div class="text-center">
										<span class="text-xs sm:text-sm font-semibold text-gray-800 dark:text-gray-200 leading-tight block">{stage.name}</span>
										<span class="text-[10px] sm:text-xs text-gray-400">{stage.prompts.length} prompts</span>
									</div>
								</button>
								{#if idx < currentWorkflow.workflow.stages.length - 1}
									<div class="hidden sm:block w-8 h-0.5 bg-gray-300 dark:bg-gray-600 flex-shrink-0 -mt-6"></div>
								{/if}
							{/each}
						</div>
					</div>

					<!-- Compact Prompts List - fixed height for 3 prompts, scrolls when more -->
					<div class="p-3 sm:p-4">
						{#if selectedStage}
							<div class="h-[138px] overflow-y-auto pr-2">
								<div class="space-y-1.5" in:fade={{ duration: 150 }}>
									{#each selectedStage.prompts as promptItem}
										<button 
											class="w-full px-3 py-2.5 text-left rounded-lg border border-gray-100 dark:border-gray-700/50 
												   hover:border-blue-300 dark:hover:border-blue-600 hover:bg-blue-50/50 dark:hover:bg-blue-900/20
												   transition-all group flex items-center justify-between"
											on:click={() => handleWorkflowPromptClick(selectedStage, promptItem)}
										>
											<span class="text-sm font-medium text-gray-700 dark:text-gray-300 group-hover:text-blue-600 dark:group-hover:text-blue-400">{promptItem.title}</span>
											<div class="flex items-center gap-2">
												{#if promptItem.features}
													{#each getFeatureIcons(promptItem.features) as feature}
														<span class="text-xs grayscale opacity-60" title={feature.label}>{feature.icon}</span>
													{/each}
												{/if}
												<svg class="w-4 h-4 text-gray-300 group-hover:text-blue-500 transition-colors" fill="none" stroke="currentColor" viewBox="0 0 24 24">
													<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
												</svg>
											</div>
										</button>
									{/each}
								</div>
							</div>
						{/if}
					</div>
				</div>
			{:else}
				<!-- Form for selected prompt -->
				<div class="p-3 sm:p-4 overflow-y-auto max-h-[70vh]">
					<button 
						class="flex items-center gap-2 text-sm text-gray-500 hover:text-gray-700 dark:hover:text-gray-300 mb-4"
						on:click={() => { selectedStagePrompt = null; inputVariables = []; currentFeatures = null; }}
					>
						<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" /></svg>
						Back to prompts
					</button>

					<div class="flex items-center gap-3 mb-4">
						<div class="w-10 h-10 rounded-xl flex items-center justify-center bg-gradient-to-br {selectedStagePrompt.stage.color}">
							<span class="text-xl">{selectedStagePrompt.stage.icon}</span>
						</div>
						<div>
							<p class="text-xs text-gray-500 dark:text-gray-400">{selectedStagePrompt.stage.name}</p>
							<h3 class="font-semibold text-gray-800 dark:text-gray-100">{selectedStagePrompt.prompt.title}</h3>
							{#if currentFeatures}
								<div class="flex items-center gap-1 mt-0.5">
									{#each getFeatureIcons(currentFeatures) as feature}
										<span class="text-[10px] px-1 py-0.5 rounded bg-gray-100 dark:bg-gray-700 grayscale opacity-70">{feature.icon} {feature.label}</span>
									{/each}
								</div>
							{/if}
						</div>
					</div>

					<form on:submit|preventDefault={handleWorkflowFormSubmit} class="space-y-3">
						{#each inputVariables as variable}
							<div class="space-y-1">
								<label class="block text-sm font-medium text-gray-700 dark:text-gray-300">{formatLabel(variable.name)}{#if variable.required}<span class="text-red-500">*</span>{/if}</label>
								{#if variable.type === 'textarea'}
									<textarea bind:value={variable.value} placeholder={variable.placeholder} required={variable.required} rows="3" class="w-full px-3 py-2 text-sm rounded-lg border border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-700 text-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500/50 resize-none" />
								{:else if variable.type === 'select'}
									<select bind:value={variable.value} required={variable.required} class="w-full px-3 py-2 text-sm rounded-lg border border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-700 text-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500/50">
										<option value="">Select...</option>
										{#each variable.options as option}<option value={option}>{option}</option>{/each}
									</select>
								{:else}
									<input type="text" bind:value={variable.value} placeholder={variable.placeholder} required={variable.required} class="w-full px-3 py-2 text-sm rounded-lg border border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-700 text-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500/50" />
								{/if}
							</div>
						{/each}
						<div class="flex justify-end gap-2 pt-4">
							<button type="button" class="px-4 py-2 text-sm rounded-lg text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700" on:click={() => { selectedStagePrompt = null; inputVariables = []; currentFeatures = null; }}>Cancel</button>
							<button type="submit" class="px-4 py-2 text-sm rounded-lg bg-blue-500 text-white hover:bg-blue-600 font-medium">
								{currentAutoSubmit ? 'Run Prompt' : 'Load Prompt'}
							</button>
						</div>
					</form>
				</div>
			{/if}
		</div>
	</div>
{/if}

<!-- Global Tooltip Portal -->
{#if activeHelpTooltip && tooltipContent}
	<div 
		class="fixed z-[99999] px-3 py-2 bg-gray-800 text-white text-xs rounded-lg shadow-xl max-w-[280px] pointer-events-none"
		style="left: {tooltipPosition.x}px; top: {tooltipPosition.y}px; transform: translate(-50%, -100%);"
		in:fade={{ duration: 100 }}
	>
		<div class="whitespace-pre-wrap leading-relaxed">{tooltipContent}</div>
		<div class="absolute left-1/2 -translate-x-1/2 top-full w-0 h-0 border-l-[6px] border-l-transparent border-r-[6px] border-r-transparent border-t-[6px] border-t-gray-800"></div>
	</div>
{/if}

<style>
	.capability-card { backdrop-filter: blur(8px); }
	.scrollbar-hide::-webkit-scrollbar { display: none; }
	.scrollbar-hide { -ms-overflow-style: none; scrollbar-width: none; }
</style>