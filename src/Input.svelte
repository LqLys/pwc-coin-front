<script>
    import {utils} from "ethers";

    export let ticket = 0;
    export let action;
    let price = utils.formatEther(ticket.price.toString());
</script>

<button class="btn btn-primary" on:click={action(ticket.id, price)}>Set price</button>
<input style="width:100px" type="number" placeholder="amount" bind:value={price}>
