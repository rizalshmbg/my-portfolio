<script lang="ts">
	interface Props {
		onNavigate: (id: string) => void;
	}

	let { onNavigate }: Props = $props();
</script>

<section class="home page-container">
	<div class="home__hero-content">
		<div class="home__intro">
			<p class="home__greeting">HELLO, I'M</p>
			<h1 class="home__full-name">RIZAL SIHOMBING</h1>
			<h2 class="home__job-title">Full-Stack Web Developer</h2>
			<p class="home__description">I build thoughtful, interactive digital experiences.</p>
		</div>

		<div class="home__hero-actions">
			<button type="button" class="btn btn--primary" onclick={() => onNavigate('experience')}>
				<span class="btn__text">Explore My Work</span>
				<span class="btn__arrow" aria-hidden="true">→</span>
			</button>

			<button type="button" class="btn btn--secondary" onclick={() => onNavigate('contact')}>
				<span class="btn__text">Get in Touch</span>
				<span class="btn__arrow" aria-hidden="true">→</span>
			</button>
		</div>
	</div>

	<div class="home__scroll-indicator" aria-hidden="true">
		<span class="home__scroll-line"></span>
		<span class="home__scroll-text">SCROLL TO EXPLORE</span>
	</div>
</section>

<style lang="scss">
	// ==========================================
	// MOBILE
	// ==========================================
	.home {
		position: relative;
		display: flex;
		flex-direction: column;
		justify-content: center;
		width: 100%;
		min-height: 100%;

		&__hero-content {
			position: relative;
			width: 100%;
		}

		&__intro {
			margin-bottom: 1rem;
		}

		&__greeting {
			margin-bottom: 0.5rem;
			font-size: 0.7rem;
			font-weight: 500;
			letter-spacing: 0.25em;
			line-height: 1.5;
			color: var(--palette-brand);
		}

		&__full-name {
			/* margin-bottom: 0.5rem; */
			font-family: 'Maharlika-Regular', 'Montserrat-Regular', sans-serif;
			font-size: clamp(2rem, 8vw, 3rem);
			line-height: 1.1;
			letter-spacing: -0.04em;
			overflow-wrap: anywhere;
			/* color: var(--palette-brand); */
		}

		&__job-title {
			margin-bottom: 0.5rem;
			font-size: clamp(1.25rem, 5vw, 1.75rem);
			font-weight: 400;
			line-height: 1.4;
			color: var(--palette-brand);
		}

		&__description {
			max-width: 30rem;
			margin-bottom: 0;
			font-size: 0.875rem;
			font-weight: 400;
			color: var(--theme-text-secondary);
		}

		&__hero-actions {
			display: flex;
			flex-direction: column;
			align-items: flex-start;
			gap: 0.875rem;
		}

		&__scroll-indicator {
			display: flex;
			align-items: center;
			gap: 0.875rem;
			margin-top: 3rem;
			color: var(--theme-text-primary);
		}

		&__scroll-line {
			display: block;
			width: 2.5rem;
			height: 1px;
			background-color: var(--palette-brand);
			transform-origin: left center;
			animation: scroll-line-pulse 2s ease-in-out infinite;
		}

		&__scroll-text {
			font-size: 0.625rem;
			font-weight: 500;
			letter-spacing: 0.2em;
			line-height: 1.5;
		}

		// ==========================================
		// TABLET — 768px AND ABOVE
		// ==========================================
		@media (min-width: 767.98px) {
			&__greeting {
				font-size: 0.75rem;
			}

			&__full-name {
				font-size: clamp(3rem, 6vw, 4.5rem);
			}

			&__job-title {
				font-size: clamp(1.65rem, 3vw, 2.25rem);
			}

			&__description {
				font-size: 1rem;
			}

			&__hero-actions {
				flex-direction: row;
				align-items: center;
				gap: 1rem;
			}

			&__scroll-indicator {
				margin-top: 4rem;
			}
		}

		// ==========================================
		// DESKTOP — 1024px AND ABOVE
		// ==========================================
		@media (min-width: 1023.98px) {
			&__greeting {
				margin-bottom: 1rem;
			}

			&__full-name {
				font-size: clamp(4rem, 5vw, 5.5rem);
			}

			&__job-title {
				font-size: clamp(2rem, 2.5vw, 2.75rem);
			}

			&__scroll-indicator {
				margin-top: 4.5rem;
			}

			&__scroll-line {
				width: 4rem;
			}
		}
	}

	.btn {
		display: inline-flex;
		align-items: center;
		justify-content: center;
		gap: 0.75rem;
		width: 100%;
		min-height: 2.875rem;
		padding: 0.75rem 1.25rem;
		border: 1px solid transparent;
		border-radius: 0.25rem;
		font: inherit;
		font-size: 0.8125rem;
		font-weight: 500;
		line-height: 1.5;
		text-align: center;
		text-decoration: none;
		cursor: pointer;
		transition:
			background-color 0.3s ease,
			border-color 0.3s ease,
			color 0.3s ease,
			transform 0.3s ease;

		&__text {
			display: inline-block;
		}

		&__arrow {
			display: inline-block;
			font-size: 1.125rem;
			line-height: 1;
			transition: transform 0.3s ease;
		}

		// Primary button
		&--primary {
			background-color: var(--palette-brand);
			border-color: var(--palette-brand);
			color: #171717;
		}

		// Secondary button
		&--secondary {
			background-color: rgba(255, 255, 255, 0.04);
			border-color: rgba(245, 245, 245, 0.35);
			color: #f5f5f5;
		}

		// ==========================================
		// TABLET — 768px AND ABOVE
		// ==========================================
		@media (min-width: 767.98px) {
			width: auto;
			min-width: 10.5rem;
			min-height: 3rem;
			padding: 0.875rem 1.375rem;
			font-size: 0.875rem;
		}

		// Hover effects only on devices that support hover
		@media (hover: hover) and (pointer: fine) {
			&:hover {
				transform: translateY(-2px);
			}

			&:hover &__arrow {
				transform: translateX(4px);
			}

			&--primary:hover {
				background-color: var(--palette-brand-hover);
				border-color: var(--palette-brand-hover);
			}

			&--secondary:hover {
				color: #10141d;
				background-color: #f7f4ef;
				border-color: #f7f4ef;
			}
		}

		// Keyboard accessibility
		&:focus-visible {
			outline: 2px solid #d6b779;
			outline-offset: 4px;
		}

		// Reduced motion
		@media (prefers-reduced-motion: reduce) {
			transition: none;

			&__arrow {
				transition: none;
			}
		}
	}

	// ==========================================
	// KEYFRAMES
	// ==========================================
	@keyframes scroll-line-pulse {
		0%,
		100% {
			transform: scaleX(0.6);
			opacity: 0.5;
		}

		50% {
			transform: scaleX(1);
			opacity: 1;
		}
	}
</style>
