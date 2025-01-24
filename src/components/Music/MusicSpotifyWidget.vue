<template>
	<div class="spotify-widget-wrapper" v-if="cp != null">
		<div class="spotify-widget" :class="{'light': theme.theme, 'dark': !theme.theme}">
			<div class="track">
				<img class="cover" :src="cp.track.art_url" alt="Track/Album Cover" draggable="false">

				<div class="details">
					<span class="title">
						{{ cp.track.song_name }}
					</span>

					<span class="artist">
						{{ cp.track.artist }}
					</span>
				</div>
			</div>

			<span class="progress"></span>
		</div>
	
		<button class="widget-btn" @click="open_on_spotify">Play</button>
	</div>

	<div class="spotify-widget-wrapper spotify-widget-null" v-else />
</template>

<script setup lang="ts">
// Libraries
import { ref, watch, onUpdated, onMounted } from "vue";
import ProgressBar from 'progressbar.js';

// Spotify Helper
import { Spotify, SpotifyError, CurrentlyPlaying } from "@/lib/spotify-helper";
// Stores
import { useSpotify } from "@/store/spotify";
import { useTheme } from "@/store/theme";

// Setup Stoeres
const theme = useTheme();
const spotify = useSpotify();

// Setup Refs
const cp = ref<CurrentlyPlaying | null>(null);

const progress_bar = ref<any | null>(null);
const progress_bar_config = {
	strokeWidth: 2,
	color: '#1DB954',
	trailWidth: 2,
	easing: 'linear',
	duration: 1000,
};

// Variables
const trail_light = "#61605f46"
const trail_dark = "#1d1c1a32"

// Normalize a int range between 0 and 1
function normalize (val: number, max: number, min: number): number { return (val - min) / (max - min); }

// Open Track on Spotify
function open_on_spotify(): void {
	if (cp.value != null) {
		window.open(cp.value.track.song_url, '_blank');
	}
}

// Import env variables for spotify
const sp_client_id = import.meta.env.VITE_SPOTIFY_CLIENT_ID;
const sp_client_secret = import.meta.env.VITE_SPOTIFY_CLIENT_SECRET;
const sp_refresh_token = import.meta.env.VITE_SPOTIFY_REFRESH_TOKEN;

// Create a new spotify instance
const client = new Spotify(sp_client_id, sp_client_secret, sp_refresh_token);

// Updating the spotify store
onMounted(() => {
	setInterval(async () => {
		// Get currently playing
		const cp = await client.get_currently_playing();

		// Check Error
		if (cp instanceof SpotifyError) {
			if (cp.message === "Error refreshing access token") {
				// Refresh token
				await client.refresh_access_token();
			}
			return;
		}

		// Update store
		spotify.set_currently_playing(cp);
	}, 1000);
})

// Watch for changes in the spotify store
watch(spotify, (new_val) => {
	// Check if the user is listening to music
	if (new_val.currently_playing != null) {
		// Check if the song is playing else set current playing to null
		if (new_val.currently_playing.is_playing) {
			// Update the current playing
			cp.value = new_val.currently_playing;
			// Shorten the song name if it is too long
			if (cp.value.track.song_name.length > 20) {
				let trimmed = cp.value.track.song_name.substring(0, 20)
				// remove the last space if there is one
				if (trimmed[trimmed.length - 1] === " ") {
					trimmed = trimmed.substring(0, trimmed.length - 1);
				}
				// add ... to the end
				cp.value.track.song_name = trimmed + "..."
			}
		}
		else {
			// Set to null
			cp.value = null;
		}
	}
});

// Watch for theme changes
watch(theme, (new_val) => {
	document.documentElement.style.setProperty('--spotify-trail-color', new_val.theme ? trail_dark : trail_light);
}, { immediate: true });

onUpdated(() => {
	// Update the progress bar
	if (cp.value != null) {
		// Check if the progress bar is null
		if (progress_bar.value == null) {
			// Create a new progress bar
			progress_bar.value = new ProgressBar.Line('.progress', progress_bar_config);
		}

		// Update the progress bar
		progress_bar.value.animate(normalize(cp.value.progress, cp.value.duration, 0));
	}
	else {
		// Destroy the progress bar and set it to null
		if (progress_bar.value != null) {
			progress_bar.value.destroy();
			progress_bar.value = null;
		}
	}
})
</script>

<style lang="scss">	
@import '@/styles/variables.scss';

.spotify-widget-wrapper {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;

	.spotify-widget {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		box-sizing: border-box;
	
		width: 100%;
		height: 100%;

		padding: 15px;
		border-radius: 15px;

		.spotify-widget-null {
			padding: 0;
		}
	
		.track {
			display: flex;
			align-items: center;
			justify-content: center;
	
			.cover {
				width: 40%;
				border-radius: 15px;	
			}
	
			.details {
				display: flex;
				flex-direction: column;
				margin-left: 15px;
	
				width: 60%;
	
				.title {
					font-size: 1.5em;
					font-weight: 600;
				}
	
				.artist {
					font-size: 1em;
					font-weight: 400;
				}
			}
		}
	
		.progress {
			width: 100%;
	
			svg {
				width: 100%;
				margin-top: 10px;
				border-radius: 2em;
	
				path {
					stroke-linecap: round;
				}

				path:first-child {
					stroke: var(--spotify-trail-color);
				}
			}
		}
	
		&.light {
			color: $light-foreground;
			background: $light-secondary;
		}
	
		&.dark {
			color: $dark-foreground;
			background: $dark-secondary;
		}
	}

	.widget-btn {
		font-family: inherit;
		font-size: 18px;
		font-weight: 400;

		width: 30%;
		height: 3em;
		margin-top: 10px;

		border: none;
		border-radius: 4em;

		color: $light-background;
		background-color: $accent-purple;

		transition: transform 0.2s ease-in-out;

		&:hover {
			cursor: pointer;
			transform: scale(1.05);
		}
	}
}

// TODO: Do responsive separately in views
</style>