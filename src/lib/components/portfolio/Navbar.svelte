<script lang="ts">
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

<header class="navbar" class:light={theme === 'light'}>
	<div class="navbar-inner">
		<!-- Logo -->
		<button type="button" class="logo" onclick={() => navigateTo('home')} aria-label="Go to Home">
			RS<span>.</span>
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
</header>

<style lang="scss">
	.navbar {
		position: absolute;
		inset: 0 0 auto;
		z-index: 50;
		box-sizing: border-box;
		width: 100%;
		padding: 20px 6%;
		color: #f7f4ef;

		&.light {
			color: #202631;

			.logo {
				color: #202631;
			}

			.nav-link,
			.mobile-nav-link {
				color: #202631;

				&:hover,
				&.active {
					color: #a56a08;
				}
			}

			.nav-link::after {
				background: #d99b32;
			}

			.theme-toggle,
			.menu-toggle {
				border-color: rgb(32 38 49 / 30%);
				color: #8b5b0b;
				background: rgb(255 255 255 / 55%);
			}

			.mobile-nav {
				border-color: rgb(32 38 49 / 12%);
				background: rgb(248 246 240 / 96%);
			}

			.mobile-nav-link + .mobile-nav-link {
				border-color: rgb(32 38 49 / 10%);
			}
		}
	}

	.navbar-inner {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 24px;
		width: 100%;
		max-width: 1600px;
		margin-inline: auto;
	}

	.logo {
		flex-shrink: 0;
		padding: 0;
		border: 0;
		background: transparent;
		color: #fff;
		font-family: Georgia, 'Times New Roman', serif;
		font-size: clamp(1.25rem, 2.2vw, 2.25rem);
		font-weight: 700;
		cursor: pointer;

		span {
			color: #ffc65c;
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
		color: #f7f4ef;
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
			background: #ffc65c;
			content: '';
			transform: scaleX(0);
			transform-origin: left;
			transition: transform 200ms ease;
		}

		&:hover,
		&.active {
			color: #ffc65c;
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
		border: 1px solid rgb(255 255 255 / 35%);
		border-radius: 50%;
		background: rgb(255 255 255 / 5%);
		color: #ffc65c;
		cursor: pointer;
		transition:
			background-color 200ms ease,
			border-color 200ms ease,
			transform 200ms ease;

		&:focus-visible {
			outline: 2px solid #ffc65c;
			outline-offset: 4px;
		}
	}

	.theme-toggle {
		svg {
			width: 20px;
			height: 20px;
		}

		&:hover {
			border-color: #ffc65c;
			background: rgb(255 198 92 / 12%);
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
		position: absolute;
		top: calc(100% - 4px);
		right: 6%;
		display: flex;
		flex-direction: column;
		width: min(280px, 88vw);
		padding: 8px 16px;
		border: 1px solid rgb(255 255 255 / 12%);
		border-radius: 12px;
		background: rgb(16 20 29 / 96%);
		box-shadow: 0 16px 40px rgb(0 0 0 / 25%);
		backdrop-filter: blur(16px);
		opacity: 0;
		visibility: hidden;
		pointer-events: none;
		transform: translateY(-8px);
		transition:
			opacity 180ms ease,
			transform 180ms ease,
			visibility 180ms ease;

		&.visible {
			opacity: 1;
			visibility: visible;
			pointer-events: auto;
			transform: translateY(0);
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
		background: transparent;
		color: #f7f4ef;
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
			border-top: 1px solid rgb(255 255 255 / 10%);
		}

		&:hover,
		&.active {
			color: #ffc65c;
		}

		&:hover svg {
			transform: translateX(3px);
		}
	}

	@media (max-width: 900px) {
		.navbar {
			padding-inline: 5%;
		}

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

		.mobile-nav {
			right: 5%;
		}
	}

	@media (max-width: 480px) {
		.navbar {
			padding: 18px 5%;
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

		.mobile-nav {
			right: 5%;
			width: min(280px, 90vw);
		}
	}

	@media (prefers-reduced-motion: reduce) {
		.nav-link,
		.nav-link::after,
		.theme-toggle,
		.menu-toggle,
		.menu-toggle span,
		.mobile-nav,
		.mobile-nav-link svg {
			transition: none;
		}
	}
</style>
