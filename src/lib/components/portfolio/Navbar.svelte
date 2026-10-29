<script lang="ts">
	import logoBlack from '#lib/assets/logo/logo-black.png';
	import logoWhite from '#lib/assets/logo/logo-white.png';

	interface Props {
		activeSection: string;
		onNavigate: (id: string) => void;
		theme: 'dark' | 'light';
		onToggleTheme: () => void;
	}

	let { activeSection, onNavigate, theme, onToggleTheme }: Props = $props();

	let menuOpen = $state(false);

	const navItems = [
		{ id: 'home', label: 'Home' },
		{ id: 'about', label: 'About' },
		{ id: 'experience', label: 'Experience' },
		{ id: 'contact', label: 'Contact' }
	];

	function navigateTo(id: string) {
		onNavigate(id);
		menuOpen = false;
	}
</script>

<header class="navbar">
	<div class="page-container">
		<div class="navbar-inner">
			<!-- Logo -->
			<button type="button" class="logo" onclick={() => navigateTo('home')} aria-label="Go to Home">
				<!-- RS<span>.</span> -->
				<img src={theme === 'dark' ? logoWhite : logoBlack} alt="Logo" />
			</button>

			<div class="navbar-actions">
				<!-- Navigasi desktop -->
				<nav class="desktop-nav" aria-label="Main navigation">
					{#each navItems as item (item.id)}
						<button
							type="button"
							class="nav-link"
							class:active={activeSection === item.id}
							aria-current={activeSection === item.id ? 'page' : undefined}
							onclick={() => navigateTo(item.id)}
						>
							{item.label}
						</button>
					{/each}
				</nav>

				<!-- Toggle tema -->
				<button
					type="button"
					class="theme-toggle"
					onclick={onToggleTheme}
					aria-label={theme === 'dark' ? 'Switch to light theme' : 'Switch to dark theme'}
					title={theme === 'dark' ? 'Light mode' : 'Dark mode'}
				>
					{#if theme === 'dark'}
						<svg
							viewBox="0 0 24 24"
							fill="none"
							stroke="currentColor"
							stroke-width="1.8"
							stroke-linecap="round"
							stroke-linejoin="round"
							aria-hidden="true"
						>
							<circle cx="12" cy="12" r="4" />
							<path
								d="M12 2v2m0 16v2M4.93 4.93l1.42 1.42
							m11.3 11.3 1.42 1.42M2 12h2m16 0h2
							M4.93 19.07l1.42-1.42m11.3-11.3 1.42-1.42"
							/>
						</svg>
					{:else}
						<svg
							viewBox="0 0 24 24"
							fill="none"
							stroke="currentColor"
							stroke-width="1.8"
							stroke-linecap="round"
							stroke-linejoin="round"
							aria-hidden="true"
						>
							<path
								d="M20.9 13A9 9 0 0 1 11 3.1
							9 9 0 1 0 20.9 13Z"
							/>
						</svg>
					{/if}
				</button>

				<!-- Hamburger toggle: tablet dan mobile -->
				<button
					type="button"
					class="menu-toggle"
					class:opened={menuOpen}
					onclick={() => (menuOpen = !menuOpen)}
					aria-label={menuOpen ? 'Close navigation menu' : 'Open navigation menu'}
					aria-expanded={menuOpen}
					aria-controls="mobile-navigation"
				>
					<span></span>
					<span></span>
					<span></span>
				</button>
			</div>
		</div>

		<!-- Navigasi mobile -->
		<nav
			id="mobile-navigation"
			class="mobile-nav"
			class:visible={menuOpen}
			aria-label="Mobile navigation"
			inert={!menuOpen}
		>
			<!-- Header khusus mobile -->
			<div class="mobile-nav-header">
				<!-- Tombol tutup -->
				<button
					type="button"
					class="mobile-close"
					onclick={() => (menuOpen = false)}
					aria-label="Close navigation menu"
				>
					<svg
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="2"
						stroke-linecap="round"
						stroke-linejoin="round"
						aria-hidden="true"
					>
						<path d="M18 6 6 18M6 6l12 12" />
					</svg>
				</button>
			</div>

			<!-- Daftar navigasi -->
			{#each navItems as item (item.id)}
				<button
					type="button"
					class="mobile-nav-link"
					class:active={activeSection === item.id}
					aria-current={activeSection === item.id ? 'page' : undefined}
					onclick={() => navigateTo(item.id)}
					tabindex={menuOpen ? 0 : -1}
				>
					<span>{item.label}</span>
					<svg
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="1.8"
						stroke-linecap="round"
						stroke-linejoin="round"
						aria-hidden="true"
					>
						<path d="M5 12h14m-6-6 6 6-6 6" />
					</svg>
				</button>
			{/each}
		</nav>
	</div>
</header>

<style lang="scss">
	.navbar {
		position: absolute;
		inset: 0 0 auto;
		z-index: 50;
		box-sizing: border-box;
		width: 100%;
		padding: 20px 0;

		background: var(--glass-background-transparent);
		border-bottom: 1px solid var(--glass-border);
		backdrop-filter: blur(1px) saturate(100%);
		-webkit-backdrop-filter: blur(1px) saturate(100%);
	}

	.navbar-inner {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 24px;
		width: 100%;
		margin-inline: auto;
	}

	.logo {
		display: inline-flex;
		align-items: center;
		justify-content: center;
		flex-shrink: 0;
		padding: 0;
		border: 0;
		background: transparent;
		font-family: 'Maharlika-Regular', 'Georgia', serif;
		font-size: clamp(1.25rem, 2.2vw, 2.25rem);
		font-weight: 700;
		cursor: pointer;

		/* span {
			color: var(--palette-brand);
		} */

		img {
			display: block;
			width: 32px;
			height: 32px;
			object-fit: contain;
		}
	}

	.navbar-actions {
		display: flex;
		align-items: center;
		gap: clamp(12px, 2vw, 32px);
	}

	.desktop-nav {
		display: flex;
		align-items: center;
		gap: clamp(20px, 3vw, 48px);
	}

	.nav-link {
		position: relative;
		padding: 8px 0;
		border: 0;
		background: transparent;
		font: inherit;
		font-size: clamp(0.8rem, 1vw, 1rem);
		white-space: nowrap;
		cursor: pointer;
		transition: color 200ms ease;

		&::after {
			position: absolute;
			right: 0;
			bottom: 0;
			left: 0;
			height: 3px;
			border-radius: 4px;
			background: var(--palette-brand);
			content: '';
			transform: scaleX(0);
			transform-origin: left;
			transition: transform 200ms ease;
		}

		&:hover {
			color: var(--palette-brand-hover);
		}

		&.active {
			color: var(--palette-brand);
		}

		&.active::after {
			transform: scaleX(1);
		}
	}

	.theme-toggle,
	.menu-toggle {
		display: inline-flex;
		flex-shrink: 0;
		align-items: center;
		justify-content: center;
		width: 40px;
		height: 40px;
		padding: 0;
		border: 1px solid var(--theme-border);
		border-radius: 50%;
		background: var(--glass-background-transparent);
		backdrop-filter: blur(0px) saturate(160%);
		color: var(--palette-brand);
		cursor: pointer;
		transition:
			color 200ms ease,
			background-color 200ms ease,
			border-color 200ms ease,
			transform 200ms ease;

		&:focus-visible {
			outline: 2px solid var(--palette-brand);
			outline-offset: 4px;
		}
	}

	.theme-toggle {
		svg {
			width: 20px;
			height: 20px;
		}

		&:hover {
			border-color: var(--palette-brand);
			background: var(--theme-surface-hover);
		}
	}

	/* Hamburger disembunyikan pada desktop */
	.menu-toggle {
		display: none;
		position: relative;

		span {
			position: absolute;
			width: 18px;
			height: 2px;
			border-radius: 2px;
			background: currentColor;
			transition:
				transform 200ms ease,
				opacity 200ms ease;
		}

		span:nth-child(1) {
			transform: translateY(-6px);
		}

		span:nth-child(3) {
			transform: translateY(6px);
		}

		&.opened {
			display: none;

			span:nth-child(1) {
				transform: rotate(45deg);
			}

			span:nth-child(2) {
				opacity: 0;
			}

			span:nth-child(3) {
				transform: rotate(-45deg);
			}
		}
	}

	/* Panel navigasi mobile */
	.mobile-nav {
		position: fixed;
		top: 0;
		right: 0;
		z-index: 49;

		display: flex;
		flex-direction: column;

		width: min(280px, 85vw);
		height: 100vh;
		height: 100dvh;
		padding: 12px 24px 32px;
		box-sizing: border-box;

		border-left: 1px solid var(--theme-border);
		background: var(--glass-background);
		backdrop-filter: blur(5px) saturate(160%);
		box-shadow: -12px 0 40px rgb(0 0 0 / 12%);

		overflow-y: auto;

		transform: translateX(100%);
		visibility: hidden;
		pointer-events: none;

		transition:
			transform 350ms cubic-bezier(0.22, 1, 0.36, 1),
			visibility 350ms ease;

		&.visible {
			transform: translateX(0);
			visibility: visible;
			pointer-events: auto;
		}
	}

	/* Header khusus mobile */
	.mobile-nav-header {
		display: flex;
		align-items: center;
		justify-content: flex-end;
		margin-bottom: 12px;
	}

	.mobile-close {
		display: inline-flex;
		align-items: center;
		justify-content: center;
		width: 36px;
		height: 36px;
		padding: 0;
		border: 1px solid var(--theme-border);
		border-radius: 50%;
		background: var(--theme-surface);
		color: var(--palette-brand);
		cursor: pointer;
		transition:
			color 200ms ease,
			background-color 200ms ease,
			border-color 200ms ease,
			transform 200ms ease;

		&:hover {
			color: var(--palette-brand-hover);
			border-color: var(--palette-brand-hover);
			background: var(--theme-surface-hover);
			transform: rotate(90deg);
		}

		&:focus-visible {
			outline: 2px solid var(--palette-brand);
			outline-offset: 4px;
		}

		svg {
			width: 20px;
			height: 20px;
		}
	}

	.mobile-nav-link {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 16px;
		width: 100%;
		padding: 14px 4px;
		border: 0;
		color: var(--theme-text-primary);
		background: transparent;
		font: inherit;
		font-size: 0.95rem;
		text-align: left;
		cursor: pointer;

		svg {
			width: 18px;
			height: 18px;
			opacity: 0.6;
			transition: transform 180ms ease;
		}

		& + & {
			border-top: 1px solid var(--theme-border);
		}

		&:hover {
			color: var(--palette-brand-hover);
		}

		&.active {
			color: var(--palette-brand);
		}

		&:hover svg {
			transform: translateX(3px);
		}
	}

	@media (min-width: 901px) {
		.mobile-nav {
			display: none;
		}
	}

	@media (max-width: 900px) {
		.desktop-nav {
			gap: 20px;
		}

		/* Menu desktop diganti hamburger */
		.desktop-nav {
			display: none;
		}

		.menu-toggle {
			display: inline-flex;
		}
	}

	@media (max-width: 480px) {
		.navbar {
			padding: 18px 0;
		}

		.navbar-inner {
			gap: 12px;
		}

		.logo {
			font-size: clamp(1.1rem, 5vw, 1.4rem);
		}

		.navbar-actions {
			gap: 10px;
		}

		.theme-toggle,
		.menu-toggle {
			width: 36px;
			height: 36px;
		}

		.theme-toggle svg {
			width: 18px;
			height: 18px;
		}
	}

	@media (prefers-reduced-motion: reduce) {
		.nav-link,
		.nav-link::after,
		.theme-toggle,
		.menu-toggle,
		.menu-toggle span,
		.mobile-nav,
		.mobile-nav-link svg,
		.mobile-close {
			transition: none;
		}
	}
</style>
