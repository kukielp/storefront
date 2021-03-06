<script>
  import { checkout } from "../api";
  import Header from "../components/Header.svelte";
  import Footer from "../components/Footer.svelte";

  export let params;
  export const shopId = params.shopId;
  export const shop = params.shop;
  let error;
  let selectedShippingOption,
    shippingTotal = "";

  $: subtotal = shop.goods
    .map(good => good.price * (good.quantity ? good.quantity : 0))
    .reduce((a, b) => a + b, 0);

  $: total = subtotal + shippingTotal;

  $: {
    if (selectedShippingOption) {
      const kilosToShip = shop.goods
        .map(good => (good.quantity ? good.quantity : 0))
        .reduce((a, b) => a + b, 0);
      const boxesToShip = Math.round(
        kilosToShip / selectedShippingOption.kgPerBox + 0.5
      );
      shippingTotal = boxesToShip * selectedShippingOption.pricePerBox;
    }
  }

  const handleCheckout = async () => {
    if (subtotal == 0) {
      error = "No items have been selected";
      return;
    }
    document.querySelector("#checkoutButton").classList.add("pure-button-disabled");
    const order = {
      goods: shop.goods.filter(good => good.quantity > 0),
      shipping: selectedShippingOption.name
    };
    const response = await checkout(shopId, order);
    const json = await response.json();
    if (!response.ok) {
      const reason = json.error ? json.error : "An unknown error occurred";
      error = `Checkout failed: ${reason}`;
      return;
    }
    var stripe = Stripe(json.stripeKey);
    stripe
      .redirectToCheckout({
        sessionId: json.sessionId
      })
      .then(function(result) {
        error = result.error.message;
      });
  };
</script>

<style>
  .quantityInput {
    width: 6em;
  }

  .error {
    background-color: red;
  }

  .itemList {
    display: grid;
    grid-template-columns: auto 100px;
    align-items: center;
    align-content: space-evenly;
    border-bottom: 1px solid lightgray;
    grid-row-gap: 1.5em;
    padding-bottom: 1.5em;
  }

  .title {
    font-weight: bold;
  }

  .comment {
    font-size: 0.9em;
    font-style: italic;
  }

  .header {
    color: gray;
    font-weight: bold;
    font-size: 0.8em;
    border-bottom: 1px solid lightgray;
  }

  .totalList {
    display: grid;
    grid-template-columns: auto 100px;
    grid-row-gap: 0.5em;
    background-color: lightgray;
    padding: 2em;
    margin: 2em;
  }

  .value {
    text-align: center;
  }

  .total {
    font-weight: bold;
  }

  .item {
    font-size: 1.2em;
  }

  .quantity {
    text-align: right;
  }

  .quantityContainer {
    text-align: right;
  }

  .quantityInput {
    width: 100%;
    text-align: center;
  }

  #checkoutButton {
    padding: 1em;
    width: 10em;
    grid-column: span 2;
    margin: 0 auto;
  }
</style>

<svelte:options accessors={true} />
<main>
  <script src="https://js.stripe.com/v3/">

  </script>
  <Header shopId={shop.id} title={shop.title} subtitle={shop.subtitle} />
  {#if shop.message && !error}
    <div class="message">{shop.message}</div>
  {/if}
  {#if error}
    <div class="message error">{error}</div>
  {/if}
  <form
    class="pure-form pure-form-aligned"
    on:submit|preventDefault={handleCheckout}>
    <div class="itemList">
      <div class="header">Item</div>
      <div class="header quantity">Quantity</div>
      {#each shop.goods as good}
        <div class="item">
          <span class="title">{good.name}</span>, ${good.price}/{good.unit}
          {#if good.comment}
            <div class="comment">{good.comment}</div>
          {/if}
        </div>
        <div class="quantityContainer">
          <input
            type="number"
            class="quantityInput"
            bind:value={good.quantity}
            min="0" />
        </div>
      {/each}
    </div>
    <div class="totalList">
      {#if shop.shippingCosts}
        <div class="label">Subtotal</div>
        <div class="value">${subtotal}</div>

        <div class="label">
          Delivery to:
          <select id="shippingOption" bind:value={selectedShippingOption}>
            {#each shop.shippingCosts as shippingOption}
              <option value={shippingOption}>{shippingOption.name}</option>
            {/each}
          </select>
        </div>
        <div class="value">
          {#if shippingTotal}${shippingTotal}{/if}
        </div>
      {/if}

      <div class="label total">Total</div>
      <div class="value total">${total}</div>
      <button type="submit" class="pure-button" id="checkoutButton">
        Purchase
      </button>
    </div>
  </form>
  {#if shop.footer}
    <Footer message={shop.footer} />
  {/if}
</main>
