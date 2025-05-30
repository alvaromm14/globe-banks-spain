<script>
    export let data;

    import { format } from "d3-format";
    import { fade } from "svelte/transition"

</script>

{#if data?.pais}
    <div transition:fade>
        <h2>{data.pais}</h2>
        <h3>{data.total.toLocaleString("es-ES", { useGrouping: true })} millones de euros</h3>
    </div>
{/if}

<style>
    div {
        position: absolute;
        bottom: 30px;
        right: 0px;
        text-align: right;
    }

    h2 {
        font-size: 1.5rem;
        font-weight: 700;
        margin-bottom: 0.25rem;
    }

    h3 {
        font-size: 1.15rem;
        font-weight: 200;
    }
</style>