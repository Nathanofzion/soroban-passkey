<script lang="ts">
	import { WebAuthn } from "@darkedges/capacitor-native-webauthn";
	import base64url from "base64url";
	import { Capacitor } from "@capacitor/core";
	import { PUBLIC_horizonUrl } from "$env/static/public";
	import { Horizon, Keypair } from "@stellar/stellar-sdk";
	import { onDestroy, onMount } from "svelte";
	import { getPublicKeys } from "$lib/webauthn";
	import { handleDeploy } from "$lib/deploy";
	import { handleVoteBuild } from "$lib/vote_build";
	import { handleVoteSend } from "$lib/vote_send";
	import { getVotes } from "$lib/get_votes";
	import { fade, blur, slide, scale } from "svelte/transition";
	import { swipe, press, tap } from "svelte-gestures";
	import { Share } from "@capacitor/share";

	// TODO break up this code into components so it's not so monolithic

	let deployee: any;
	let bundlerKey: Keypair;
	let votes = {
		all_votes: {
			chicken: 0,
			egg: 0,
			chicken_percent: 0,
			egg_percent: 0,
			chicken_percent_no_source: 0,
			egg_percent_no_source: 0,
		},
		source_votes: {
			chicken: 0,
			egg: 0,
			chicken_percent: 0,
			egg_percent: 0,
		},
		total_source_votes: 0,
		total_all_votes: 0,
	};
	let loadingRegister = false;
	let loadingSign = false;

	let step = 0;
	let dotinterval: NodeJS.Timeout;
	let voteinterval: NodeJS.Timeout;
	let dots = "";
	let choice: string | null;

	onDestroy(() => {
		clearInterval(dotinterval);
		clearInterval(voteinterval);
	});

	onMount(async () => {
		setTimeout(() => (step = 1), 500);

		dotinterval = setInterval(() => {
			if (deployee) clearInterval(dotinterval);
			else if (dots.length === 3) dots = "";
			else dots += ".";
		}, 500);

		voteinterval = setInterval(() => onVotes(), 5000);

		if (localStorage.hasOwnProperty("sp:bundler")) {
			bundlerKey = Keypair.fromSecret(
				localStorage.getItem("sp:bundler")!,
			);
		} else {
			bundlerKey = Keypair.random();
			localStorage.setItem("sp:bundler", bundlerKey.secret());

			const horizon = new Horizon.Server(PUBLIC_horizonUrl);
			await horizon.friendbot(bundlerKey.publicKey()).call();
		}

		if (localStorage.hasOwnProperty("sp:deployee")) {
			deployee = localStorage.getItem("sp:deployee");
			await onVotes();
		}
	});

	const onRegister = async (type?: "signin") => {
		if (!type && deployee) {
			step++;
			return;
		}

		try {
			loadingRegister = true;

			if (type === "signin") {
				const signRes = await WebAuthn.startAuthentication({
					challenge: base64url("createchallenge"),
					rpId: Capacitor.isNativePlatform()
						? "passkey.sorobanbyexample.org"
						: undefined,
					userVerification: "discouraged",
				});

				localStorage.setItem("sp:id", signRes.id);

				// as-is signin cannot retrieve a public-key so we can only derive the contract address we cannot actually deploy the abstract account
				const { contractSalt } = await getPublicKeys(signRes);
				deployee = await handleDeploy(bundlerKey, contractSalt);
			} else {
				const registerRes = await WebAuthn.startRegistration({
					challenge: base64url("createchallenge"),
					rp: {
						id: Capacitor.isNativePlatform()
							? "passkey.sorobanbyexample.org"
							: undefined,
						name: "SoroPass",
					},
					user: {
						id: base64url("Soroban Test"),
						name: "Soroban Test",
						displayName: "Soroban Test",
					},
					authenticatorSelection: {
						requireResidentKey: false,
						residentKey:
							Capacitor.getPlatform() === "android"
								? "preferred" // `discouraged` bugs with error [34000] on Android
								: "discouraged",
						userVerification: "discouraged",
					},
					pubKeyCredParams: [{ alg: -7, type: "public-key" }],
					attestation: "none",
				});

				localStorage.setItem("sp:id", registerRes.id);

				const { contractSalt, publicKey } =
					await getPublicKeys(registerRes);
				deployee = await handleDeploy(
					bundlerKey,
					contractSalt,
					publicKey!,
				);
			}

			console.log(deployee);
			localStorage.setItem("sp:deployee", deployee);
			step++;
		} catch (error) {
			console.error(error);
			alert(JSON.stringify(error));
		} finally {
			loadingRegister = false;
		}
	};

	const onSign = async () => {
		try {
			loadingSign = true;

			let { authTxn, authHash, lastLedger } = await handleVoteBuild(
				bundlerKey,
				deployee,
				choice === "chicken",
			);

			const signRes = await WebAuthn.startAuthentication({
				challenge: base64url(authHash),
				rpId: Capacitor.isNativePlatform()
					? "passkey.sorobanbyexample.org"
					: undefined,
				allowCredentials: localStorage.hasOwnProperty("sp:id")
					? [
							{
								id: localStorage.getItem("sp:id")!,
								type: "public-key",
							},
						]
					: undefined,
				userVerification: "discouraged",
			});

			await handleVoteSend(bundlerKey, authTxn, lastLedger, signRes);
			await onVotes();
			step++;
		} catch (error) {
			console.error(error);
			alert(JSON.stringify(error));
		} finally {
			loadingSign = false;
		}
	};

	const onVotes = async () => {
		if (bundlerKey && deployee) {
			votes = await getVotes(bundlerKey, deployee);
			console.log(votes);
		}
	};

	function truncateAccount(account: string) {
		return `${account.slice(0, 5)}...${account.slice(-5)}`;
	}

	function swipeHandler(event: CustomEvent) {
		if (event.detail.direction === "right") goLeft();
		else if (event.detail.direction === "left") goRight();
	}
	function tapHandler(event: CustomEvent) {
		if (
			!["div", "h1", "p"].includes(
				event.detail.target.tagName.toLowerCase(),
			)
		)
			return;
		else if (
			document.querySelector("#soropass")?.clientWidth! / 2 >
			event.detail.x
		)
			goLeft();
		else goRight();
	}

	function goLeft() {
		if (!(step <= 1)) step--;
	}

	function goRight() {
		if (
			!(
				step >= 14 ||
				(step === 5 && !deployee) ||
				(step === 10 && !choice && !votes?.total_source_votes) ||
				(step === 11 && !votes?.total_source_votes)
			)
		)
			step++;
	}

	async function share() {
		const { value } = await Share.canShare();

		if (value) {
			await Share.share({
				title: "Share SoroPass",
				text: "Check out this blockchain experience powered by your face or fingers!",
				url: "https://passkey.sorobanbyexample.org/",
				dialogTitle: `${choice === "chicken"} ? 'Chicken 🐔' : 'Egg 🥚'} people unite!`,
			});
		} else {
			window.open(
				`https://twitter.com/intent/tweet?text=${encodeURIComponent("Check out this blockchain experience powered by your face or fingers!")}&url=${encodeURIComponent("https://passkey.sorobanbyexample.org/")}`,
			);
		}
	}

	function resetAll() {
		localStorage.removeItem("sp:id");
		localStorage.removeItem("sp:bundler");
		localStorage.removeItem("sp:deployee");
		window.location.reload();
	}
</script>

<div
	id="soropass"
	class="relative w-full flex flex-col items-center justify-center h-dvh px-2 select-none overflow-hidden bg-violet-800 {!Capacitor.isNativePlatform()
		? 'max-h-[800px] max-w-[500px] py-2'
		: null} {loadingRegister || loadingSign ? 'pointer-events-none' : null}"
	use:swipe={{ timeframe: 300, minSwipeDistance: 100, touchAction: "pan-y" }}
	use:tap={{ timeframe: 300 }}
	on:swipe={swipeHandler}
	on:tap={tapHandler}
>
	{#if step > 0}
		<div
			class="flex w-full items-center justify-between"
			use:press={{ timeframe: 1000, triggerBeforeFinished: true }}
			on:press={resetAll}
		>
			<div
				class="flex items-center origin-left"
				transition:scale={{
					duration: 500,
					delay: 250,
					opacity: 0,
					start: 0.8,
				}}
			>
				<svg
					class="stroke-violet-800 rounded-full border-2 border-yellow-500 {deployee
						? 'bg-yellow-500'
						: null}"
					viewBox="0 0 15 15"
					fill="none"
					xmlns="http://www.w3.org/2000/svg"
					width="25"
					height="25"><path d="M4 7.5L7 10l4-5"></path></svg
				>

				{#if deployee}
					<span class="font-mono text-sm ml-2"
						>{truncateAccount(deployee)}</span
					>
				{:else}
					<span class="font-mono text-sm ml-2">{dots}</span>
				{/if}
			</div>

			<button
				class="flex items-center font-mono text-xs uppercase origin-right py-2 pl-2 {deployee
					? null
					: 'invisible'}"
				transition:scale={{
					duration: 500,
					delay: 250,
					opacity: 0,
					start: 0.8,
				}}
				on:click={resetAll}
			>
				Restart
				<svg
					class="stroke-yellow-500 ml-2"
					viewBox="0 0 15 15"
					fill="none"
					xmlns="http://www.w3.org/2000/svg"
					width="15"
					height="15"
					><path
						d="M7.5 14.5A7 7 0 013.17 2M7.5.5A7 7 0 0111.83 13m-.33-3v3.5H15M0 1.5h3.5V5"
					></path></svg
				>
			</button>
		</div>
	{/if}

	<div class="w-full relative text-center text-3xl font-bold my-auto">
		{#if step === 1}
			<h1
				class="absolute w-full top-0 -translate-y-1/2"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				Welcome to a passkey powered blockchain experience
			</h1>
		{/if}

		{#if step === 2}
			<div
				class="absolute w-full top-0 -translate-y-1/2"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				<h1
					class=""
					in:fade={{ delay: 0, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Fully non-custodial
				</h1>
				<h1
					class=""
					in:fade={{ delay: 500, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					But also
				</h1>
				<h1
					class=""
					in:fade={{ delay: 1250, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Entirely convenient
				</h1>
			</div>
		{/if}

		{#if step === 3}
			<div
				class="absolute w-full top-0 -translate-y-1/2 px-3 text-left"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				<p
					in:fade={{ delay: 0, duration: 250 }}
					out:fade={{ duration: 250 }}
					class="text-xl"
				>
					You’re just a single tap away from a fully featured future
					of financial freedom powered by your face or fingers.
				</p>
				<br />
				<h1
					in:fade={{ delay: 1250, duration: 250 }}
					out:fade={{ duration: 250 }}
					class=""
				>
					F passphrases
				</h1>
			</div>
		{/if}

		{#if step === 4}
			<h1
				class="absolute w-full top-0 -translate-y-1/2"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				Are you ready?
			</h1>
		{/if}

		{#if step === 5}
			<div
				class="absolute flex flex-col items-center justify-center w-full top-0 -translate-y-1/2"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				<h1
					class=""
					in:fade={{ delay: 0, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Press the button
				</h1>

				<br />

				<button
					class="relative inline-flex items-center border-2 border-yellow-500 rounded-lg p-2 bg-yellow-500/10 ring-2 ring-yellow-500/50 ring-offset-4 ring-offset-violet-800 shadow-2xl shadow-yellow-500/50 active:shadow-yellow-500/30 active:top-[2px]"
					in:fade={{ delay: 250, duration: 250 }}
					out:fade={{ duration: 250 }}
					on:click={() => onRegister()}
				>
					<svg
						viewBox="0 0 15 15"
						fill="none"
						xmlns="http://www.w3.org/2000/svg"
						width="30"
						height="30"
						><path
							d="M4 6h1V5H4v1zm6 0h1V5h-1v1zm.1 2.7a3.25 3.25 0 01-5.2 0l-.8.6c1.7 2.267 5.1 2.267 6.8 0l-.8-.6zM1 5V2.5H0V5h1zm1.5-4H5V0H2.5v1zM1 2.5A1.5 1.5 0 012.5 1V0A2.5 2.5 0 000 2.5h1zM0 10v2.5h1V10H0zm2.5 5H5v-1H2.5v1zM0 12.5A2.5 2.5 0 002.5 15v-1A1.5 1.5 0 011 12.5H0zM10 1h2.5V0H10v1zm4 1.5V5h1V2.5h-1zM12.5 1A1.5 1.5 0 0114 2.5h1A2.5 2.5 0 0012.5 0v1zM10 15h2.5v-1H10v1zm5-2.5V10h-1v2.5h1zM12.5 15a2.5 2.5 0 002.5-2.5h-1a1.5 1.5 0 01-1.5 1.5v1z"
							fill="currentColor"
						></path></svg
					>
					<div
						class="flex items-center justify-center"
						transition:slide={{
							duration: 250,
							delay: 250,
							axis: "x",
						}}
					>
						{#if loadingRegister}
							<svg
								class="mx-4"
								transition:slide={{
									duration: 150,
									delay: 0,
									axis: "x",
								}}
								xmlns="http://www.w3.org/2000/svg"
								width="30"
								height="30"
								viewBox="0 0 24 24"
								><path
									class="fill-yellow-500"
									d="M10.72,19.9a8,8,0,0,1-6.5-9.79A7.77,7.77,0,0,1,10.4,4.16a8,8,0,0,1,9.49,6.52A1.54,1.54,0,0,0,21.38,12h.13a1.37,1.37,0,0,0,1.38-1.54,11,11,0,1,0-12.7,12.39A1.54,1.54,0,0,0,12,21.34h0A1.47,1.47,0,0,0,10.72,19.9Z"
									><animateTransform
										attributeName="transform"
										dur="0.75s"
										repeatCount="indefinite"
										type="rotate"
										values="0 12 12;360 12 12"
									/></path
								></svg
							>
						{:else}
							<span
								class="mx-4 font-mono uppercase text-lg"
								transition:slide={{
									duration: 150,
									delay: 0,
									axis: "x",
								}}>Register</span
							>
						{/if}
					</div>

					<svg
						viewBox="0 0 15 15"
						fill="none"
						xmlns="http://www.w3.org/2000/svg"
						width="30"
						height="30"
						><path
							d="M12.587 3.513a6.03 6.03 0 01.818 3.745v.75c0 .788.205 1.563.595 2.247M4.483 6.508c0-.795.313-1.557.871-2.119a2.963 2.963 0 012.103-.877c.789 0 1.545.315 2.103.877.558.562.871 1.324.871 2.12v.748c0 1.621.522 3.198 1.487 4.495m-4.46-5.244v1.498A10.542 10.542 0 009.315 14M4.483 9.505A13.559 13.559 0 005.821 14m-3.643-1.498a16.63 16.63 0 01-.669-5.244V6.51a6.028 6.028 0 01.79-3.002 5.97 5.97 0 012.177-2.2 5.914 5.914 0 015.955-.004"
							stroke="currentColor"
							stroke-linecap="square"
							stroke-linejoin="round"
						></path></svg
					>
				</button>

				<button
					class="text-sm font-mono uppercase px-6 py-4 mt-6"
					on:click={() => onRegister("signin")}
					in:fade={{ delay: 500, duration: 250 }}
					out:fade={{ duration: 250 }}>Sign In</button
				>
			</div>
		{/if}

		{#if step === 6}
			<div
				class="absolute w-full top-0 -translate-y-1/2 px-3 text-left"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				<h1
					class=""
					in:fade={{ delay: 0, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					And just like that you’re in!
				</h1>

				<br />

				<p
					class="text-xl"
					in:fade={{ delay: 250, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					A singular action granting you access to a global financial
					ecosystem of hundreds of currencies, financial instruments,
					and services now all just a glance or tap away.
				</p>
			</div>
		{/if}

		{#if step === 7}
			<h1
				class="absolute w-full top-0 -translate-y-1/2"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				It took humans awhile to get here, but we did, <br /> you did. Incredible.
			</h1>
		{/if}

		{#if step === 8}
			<div
				class="absolute w-full top-0 -translate-y-1/2 px-3 text-left"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				<h1
					class=""
					in:fade={{ delay: 0, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					This is your new passkey powered blockchain account.
				</h1>
				<br />
				<pre
					class="relative flex items-center justify-between border-b-2 border-yellow-500 bg-yellow-500/10 py-2 px-4 select-text"
					in:fade={{ delay: 150, duration: 250 }}
					out:fade={{ duration: 250 }}>
					<code class="font-mono text-base"
						>{deployee?.substring(0, 28)}<br />{deployee?.substring(
							28,
						)}</code
					>

					<!-- Remove for now due to latency with block explorers indexing futurenet data -->
					<!-- <a class="flex absolute inset-0" href="https://futurenet.steexp.com/contract/{deployee}">
						<svg
							class="absolute right-4 top-1/2 -translate-y-1/2 -rotate-45 origin-center"
							viewBox="0 0 15 15"
							fill="none"
							xmlns="http://www.w3.org/2000/svg"
							width="25"
							height="25"><path d="M13.5 7.5l-4-4m4 4l-4 4m4-4H1" stroke="currentColor"></path></svg
						>
					</a> -->
				</pre>
				<br />
				<p
					class="text-xl"
					in:fade={{ delay: 300, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					You can sign whatever you want with it. And only you can.
				</p>
			</div>
		{/if}

		{#if step === 9}
			<div
				class="absolute w-full top-0 -translate-y-1/2 px-3 text-left"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				<p
					class="text-xl"
					in:fade={{ delay: 0, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Let’s demonstrate our cryptographic super powers by settling
					once and for all an age old question...
				</p>
				<br />
				<h1
					class=""
					in:fade={{ delay: 1000, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Which came first <br /> the chicken 🐔 <br /> or the egg 🥚?
				</h1>
			</div>
		{/if}

		{#if step === 10}
			<div
				class="absolute w-full top-0 -translate-y-1/2 px-3 text-left"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				<h1
					class=""
					in:fade={{ delay: 0, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Make your choice
				</h1>
				<br />
				<p
					class="text-xl"
					in:fade={{ delay: 250, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Then cryptographically sign it with your passkey powered
					blockchain account to securely record your vote for eternal,
					immutable assurance alongside all other participants.
				</p>
				<br />
				<div
					class="grid grid-cols-2 grid-rows-1 gap-4"
					in:fade={{ delay: 500, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					<button
						class="border-2 border-yellow-500 rounded-full p-3 text-[3rem] {choice ===
						'chicken'
							? 'bg-yellow-500/10 ring-2 ring-yellow-500/50 ring-offset-4 ring-offset-violet-800'
							: null}"
						on:click={() =>
							choice === "chicken"
								? (choice = null)
								: (choice = "chicken")}>🐔</button
					>
					<button
						class="border-2 border-yellow-500 rounded-full p-3 text-[3rem] {choice ===
						'egg'
							? 'bg-yellow-500/10 ring-2 ring-yellow-500/50 ring-offset-4 ring-offset-violet-800'
							: null}"
						on:click={() =>
							choice === "egg"
								? (choice = null)
								: (choice = "egg")}>🥚</button
					>
				</div>
			</div>
		{/if}

		{#if step === 11}
			<div
				class="absolute w-full top-0 -translate-y-1/2 px-3 text-left"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				<h1
					class=""
					in:fade={{ delay: 0, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					An eggcellent choice
				</h1>
				<br />
				<p
					class="text-xl"
					in:fade={{ delay: 250, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Now press once more to secure your precious opinion for all
					time for all people everywhere to settle this debate. No
					middle men, no trust, no 3rd party services. Just your
					biometrics put to work by the power of math to solve
					humanity's most difficult challenges.
				</p>
				<br />
				<button
					class="relative w-full flex items-center justify-between border-2 border-yellow-500 rounded-lg p-2 bg-yellow-500/10 ring-2 ring-yellow-500/50 ring-offset-4 ring-offset-violet-800 shadow-2xl shadow-yellow-500/50 active:shadow-yellow-500/30 active:top-[2px]"
					in:fade={{ delay: 500, duration: 250 }}
					out:fade={{ duration: 250 }}
					on:click={onSign}
				>
					<svg
						viewBox="0 0 15 15"
						fill="none"
						xmlns="http://www.w3.org/2000/svg"
						width="30"
						height="30"
						><path
							d="M4 6h1V5H4v1zm6 0h1V5h-1v1zm.1 2.7a3.25 3.25 0 01-5.2 0l-.8.6c1.7 2.267 5.1 2.267 6.8 0l-.8-.6zM1 5V2.5H0V5h1zm1.5-4H5V0H2.5v1zM1 2.5A1.5 1.5 0 012.5 1V0A2.5 2.5 0 000 2.5h1zM0 10v2.5h1V10H0zm2.5 5H5v-1H2.5v1zM0 12.5A2.5 2.5 0 002.5 15v-1A1.5 1.5 0 011 12.5H0zM10 1h2.5V0H10v1zm4 1.5V5h1V2.5h-1zM12.5 1A1.5 1.5 0 0114 2.5h1A2.5 2.5 0 0012.5 0v1zM10 15h2.5v-1H10v1zm5-2.5V10h-1v2.5h1zM12.5 15a2.5 2.5 0 002.5-2.5h-1a1.5 1.5 0 01-1.5 1.5v1z"
							fill="currentColor"
						></path></svg
					>
					<div
						class="absolute w-full flex items-center justify-center top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2"
					>
						{#if loadingSign}
							<svg
								transition:blur={{ amount: 10 }}
								xmlns="http://www.w3.org/2000/svg"
								width="30"
								height="30"
								viewBox="0 0 24 24"
								><path
									class="fill-yellow-500"
									d="M10.72,19.9a8,8,0,0,1-6.5-9.79A7.77,7.77,0,0,1,10.4,4.16a8,8,0,0,1,9.49,6.52A1.54,1.54,0,0,0,21.38,12h.13a1.37,1.37,0,0,0,1.38-1.54,11,11,0,1,0-12.7,12.39A1.54,1.54,0,0,0,12,21.34h0A1.47,1.47,0,0,0,10.72,19.9Z"
									><animateTransform
										attributeName="transform"
										dur="0.75s"
										repeatCount="indefinite"
										type="rotate"
										values="0 12 12;360 12 12"
									/></path
								></svg
							>
						{:else}
							<span
								class="absolute inset-0 flex items-center justify-center mx-4 font-mono uppercase text-lg"
								transition:blur={{ amount: 10 }}
								>Sign for <span class="text-3xl ml-2"
									>{choice === "chicken" ? "🐔" : "🥚"}</span
								></span
							>
						{/if}
					</div>

					<svg
						viewBox="0 0 15 15"
						fill="none"
						xmlns="http://www.w3.org/2000/svg"
						width="30"
						height="30"
						><path
							d="M12.587 3.513a6.03 6.03 0 01.818 3.745v.75c0 .788.205 1.563.595 2.247M4.483 6.508c0-.795.313-1.557.871-2.119a2.963 2.963 0 012.103-.877c.789 0 1.545.315 2.103.877.558.562.871 1.324.871 2.12v.748c0 1.621.522 3.198 1.487 4.495m-4.46-5.244v1.498A10.542 10.542 0 009.315 14M4.483 9.505A13.559 13.559 0 005.821 14m-3.643-1.498a16.63 16.63 0 01-.669-5.244V6.51a6.028 6.028 0 01.79-3.002 5.97 5.97 0 012.177-2.2 5.914 5.914 0 015.955-.004"
							stroke="currentColor"
							stroke-linecap="square"
							stroke-linejoin="round"
						></path></svg
					>
				</button>
			</div>
		{/if}

		{#if step === 12}
			<div
				class="absolute w-full top-0 -translate-y-1/2 px-3"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				<h1
					class=""
					in:fade={{ delay: 0, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Incredible!
				</h1>
				<br />
				<div
					class="flex items-center justify-between text-lg mb-2"
					in:fade={{ delay: 100, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					<p>Chicken</p>
					<p>Egg</p>
				</div>
				<div
					class="w-full flex items-center justify-stetch h-10"
					in:slide={{
						duration: 250,
						delay: 200,
						axis: "x",
					}}
					out:slide={{
						duration: 250,
						delay: 0,
						axis: "x",
					}}
				>
					<div class="w-full flex justify-end bg-yellow-500/10 h-10">
						<div
							class="bg-yellow-500 h-10"
							style="width: {votes?.all_votes
								.chicken_percent_no_source}%"
						></div>
						<div
							class="bg-teal-500 h-10 {votes?.source_votes.chicken
								? 'min-w-1'
								: null}"
							style="width: {votes?.source_votes
								.chicken_percent}%"
						></div>
					</div>
					<hr class="h-10 border border-violet-800" />
					<hr class="h-16 border border-yellow-500" />
					<div
						class="w-full flex justify-start bg-yellow-500/10 h-10"
					>
						<div
							class="bg-teal-500 h-10 {votes?.source_votes.egg
								? 'min-w-1'
								: null}"
							style="width: {votes?.source_votes.egg_percent}%"
						></div>
						<div
							class="bg-yellow-500"
							style="width: {votes?.all_votes
								.egg_percent_no_source}%"
						></div>
					</div>
				</div>
				<div
					class="flex items-center justify-between text-lg mt-2"
					in:fade={{ delay: 300, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					<span class="text-xs font-mono"
						>[{Number(
							votes?.all_votes.chicken_percent.toFixed(2),
						)}%]</span
					>
					<span class="text-xs font-mono"
						>[{Number(
							votes?.all_votes.egg_percent.toFixed(2),
						)}%]</span
					>
				</div>
				<br />
				<div
					class="flex items-center mt-auto"
					in:fade={{ delay: 400, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					<span class="w-2 h-2 bg-teal-500 mr-2"></span>
					<p class="font-mono text-xs text-teal-500">
						Your vote{votes.total_source_votes > 1 ? "s" : ""}
					</p>
				</div>
			</div>
		{/if}

		{#if step === 13}
			<div
				class="absolute w-full top-0 -translate-y-1/2 px-3 text-left"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				<h1
					class=""
					in:fade={{ delay: 0, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					The future is secure.
				</h1>
				<h1
					class=""
					in:fade={{ delay: 750, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					But it is also simple.
				</h1>
				<br />
				<h1
					in:fade={{ delay: 2000, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Yippee!
				</h1>
				<!-- <h1
					class=""
					in:fade={{ delay: 2000, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Yay people and their brains.
				</h1>
				<h1
					class=""
					in:fade={{ delay: 3250, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Yay God in heaven,
				</h1>
				<h1
					class=""
					in:fade={{ delay: 4250, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					who made us this way.
				</h1> -->
			</div>
		{/if}

		{#if step === 14}
			<div
				class="absolute w-full top-0 -translate-y-1/2 px-3 text-left"
				transition:scale={{
					duration: 500,
					delay: 0,
					opacity: 0,
					start: 1.5,
				}}
			>
				<h1
					class=""
					in:fade={{ delay: 0, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					SoroPass
				</h1>
				<br />
				<p
					class="text-xl"
					in:fade={{ delay: 100, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Learn more about the Stellar blockchain which powers this
					experience: <a
						class="underline"
						href="https://stellar.org/soroban"
						>stellar.org/soroban</a
					>
				</p>
				<br />
				<p
					class="text-xl"
					in:fade={{ delay: 200, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Read the code here: <a
						class="underline"
						href="https://github.com/kalepail/soroban-passkey"
						>github.com/kalepail/soroban-passkey</a
					>
				</p>
				<br />
				<p
					class="text-xl"
					in:fade={{ delay: 300, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					Join our Discord: <a
						class="underline"
						href="https://discord.com/invite/stellardev"
						>discord.com/stellardev</a
					>
				</p>
				<br />
				<button
					class="relative flex items-center justify-center border-2 border-yellow-500 rounded-full px-6 py-3 bg-yellow-500/10 ring-2 ring-yellow-500/50 ring-offset-4 ring-offset-violet-800 shadow-2xl shadow-yellow-500/50 active:shadow-yellow-500/30 active:top-[2px]"
					in:fade={{ delay: 400, duration: 250 }}
					out:fade={{ duration: 250 }}
					on:click={share}
				>
					<span class="font-mono uppercase text-base"
						>Share with your friends</span
					>
				</button>
				<br />
				<p
					class="text-xl"
					in:fade={{ delay: 500, duration: 250 }}
					out:fade={{ duration: 250 }}
				>
					{choice === "chicken" ? "Chicken 🐔" : "Egg 🥚"} people unite!
				</p>
			</div>
		{/if}
	</div>

	{#if step > 0}
		<div
			class="w-full flex items-center justify-center origin-bottom"
			transition:scale={{
				duration: 500,
				delay: 250,
				opacity: 0,
				start: 0.8,
			}}
		>
			<button
				class="w-full flex items-center justify-end relative {step <= 1
					? 'invisible pointer-events-none'
					: null} active:right-[2px]"
				on:click={() => step--}
			>
				<svg
					class="p-2"
					viewBox="0 0 15 15"
					fill="none"
					xmlns="http://www.w3.org/2000/svg"
					width="45"
					height="45"
					><path
						d="M1.5 7.5l4-4m-4 4l4 4m-4-4H14"
						stroke="currentColor"
					></path></svg
				>
			</button>
			<span class="shrink-0 mx-3 tabular-nums font-mono text-xs"
				>{step} of 14</span
			>
			{#if step >= 14}
				<button
					class="w-full flex items-center justify-start relative active:top-[2px]"
					on:click={resetAll}
				>
					<svg
						class="stroke-violet-800 bg-yellow-500 rounded-full p-2"
						viewBox="0 0 15 15"
						fill="none"
						xmlns="http://www.w3.org/2000/svg"
						width="45"
						height="45"
						><path
							d="M7.5 14.5A7 7 0 013.17 2M7.5.5A7 7 0 0111.83 13m-.33-3v3.5H15M0 1.5h3.5V5"
						></path></svg
					>
				</button>
			{:else}
				<button
					class="w-full flex items-center justify-start relative {(step ===
						5 &&
						!deployee) ||
					(step === 10 && !choice && !votes?.total_source_votes) ||
					(step === 11 && !votes?.total_source_votes)
						? 'invisible pointer-events-none'
						: null} active:left-[2px]"
					on:click={() => step++}
				>
					<svg
						class="stroke-violet-800 bg-yellow-500 rounded-full p-2"
						viewBox="0 0 15 15"
						fill="none"
						xmlns="http://www.w3.org/2000/svg"
						width="45"
						height="45"
						><path d="M13.5 7.5l-4-4m4 4l-4 4m4-4H1"></path></svg
					>
				</button>
			{/if}
		</div>
	{/if}
</div>
