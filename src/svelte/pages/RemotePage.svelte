<script lang="ts">
    import { splitScreenActive } from "../store/store";
    export let title = "Remote Frontend";
    let scale = 1.0;
    let width = 1900;
    let displayScreen = "none";

    $: if (width) withChanged();

    splitScreenActive.subscribe((value) => {
        console.log("splitScreen", value);
    });

    function withChanged() {
        if (width < 1280) scale = width / 1280;
        else scale = 1;
        console.log("withChanged", width, scale);
    }

    function isLoaded() {
        displayScreen = "block";
    }
</script>

<div bind:clientWidth={width} class="container">
    <h1 class="responsive-headline">LOADING</h1>

    <iframe
        on:load={isLoaded}
        title="remote page"
        class="responsive-iframe"
        src="https://example.com/"
        style="--scale: {scale}; --displayScreen: {displayScreen}"
    ></iframe>
</div>

<style lang="scss">
    .container {
        color: black;
        text-align: center;
        position: relative;
        width: 100%;
        overflow: hidden;
        padding-top: 100%;
    }
    .responsive-iframe {
        display: var(--displayScreen);
        position: absolute;
        top: 0;
        left: 0;
        bottom: 0;
        right: 0;
        width: 1280px;
        height: 100%;
        border: none;
        transform: scale(var(--scale));
        transform-origin: 0 0;
    }
    .responsive-headline {
        position: absolute;
        top: 0;
        left: 0;
        bottom: 0;
        right: 0;
        margin-top: 40%;
        width: 100%;
        height: 100%;
        border: none;
    }
</style>
