<template>
  <Layout>
    <div class="hero">
      <div class="hero-body">
        <div class="container has-text-centered">
          <h3 class="title">
            Cart
          </h3>
        </div>
      </div>
    </div>
    <div class="container">
      <table class="table is-fullwidth">
        <thead>
          <tr>
            <th />
            <th>Product</th>
            <th>Quantity</th>
            <th>Total</th>
            <th />
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="item in cart"
            :key="item.variantId">
            <td>
              <figure class="image is-square">
                <img
                  :src="item.image.thumbnail"
                  :alt="item.image.altText || item.title">
              </figure>
            </td>
            <td>
              {{ item.productTitle }}
              {{ item.variantTitle !== 'Default Title' ? `- ${item.variantTitle}` : '' }}
            </td>
            <td>{{ item.qty }}</td>
            <td>{{ totalPrice(item) }}</td>
            <td
              width="200"
              class="has-text-right">
              <button
                class="delete is-danger"
                @click="removeItem(item.variantId)"
                @keyup="removeItem(item.variantId)">
                <small>Remove</small>
              </button>
            </td>
          </tr>
        </tbody>
        <tfoot v-if="cart.length">
          <tr>
            <th />
            <th />
            <th />
            <th />
            <th class="has-text-right">
              <p>Cart Total: {{ cartTotal }}</p>
            </th>
          </tr>
        </tfoot>
      </table>
      <br>
      <form
        v-if="cart.length"
        @submit.prevent="checkout">
        <div class="field is-grouped is-grouped-right">
          <div class="field has-addons">
            <div class="control">
              <label
                for="email"
                class="label">
                <input
                  id="email"
                  v-model="email"
                  class="input"
                  type="email"
                  placeholder="Your email address"
                  required>
              </label>
            </div>
            <div class="control">
              <button
                :class="{'is-loading': isLoading}"
                type="submit"
                class="button is-primary">
                Checkout
              </button>
            </div>
          </div>
        </div>
      </form>
      <div
        v-else
        class="container has-text-centered">
        <p>To checkout, add some items to cart.</p>
        <br>
        <g-link
          to="/"
          class="button is-primary is-outlined">
          Browse
        </g-link>
      </div>
    </div>
  </Layout>
</template>

<script>
import currency from 'currency.js'
import gql from 'graphql-tag'
export default {
  metaInfo: {
    title: 'Your Cart'
  },
  data: () => ({ email: '', isLoading: false }),
  computed: {
    cart () { return this.$store.state.cart },
    cartTotal () {
      const total = this.cart.reduce((total, item) => total.add(currency(item.price.amount).multiply(item.qty)), currency(0, { formatWithSymbol: true, symbol: '£' }))
      return total.format()
    }
  },
  methods: {
    totalPrice ({ qty, price }) {
      return currency(price.amount, { formatWithSymbol: true, symbol: '£' }).multiply(qty).format()
    },
    async removeItem (itemId) {
      await this.$store.commit('removeFromCart', itemId)
      this.$notify({
        title: 'Item removed from cart',
        type: 'primary'
      })
    },
    async checkout () {
      const email = this.email
      if (!this.cart.length) return alert('No items in cart')
      const lineItems = this.cart.map(item => ({ quantity: item.qty, variantId: item.variantId }))

      const checkoutInput = { email, lineItems }

      try {
        this.isLoading = true
        const { data: { checkoutCreate } } = await this.$apollo.mutate({
          mutation: gql`mutation checkoutCreate($input: CheckoutCreateInput!) {
            checkoutCreate(input: $input) {
              checkout {
                id
                webUrl
              }
              checkoutUserErrors {
                code
                field
                message
              }
            }
          }
          `,
          variables: { input: checkoutInput }
        })
        if (checkoutCreate.checkoutUserErrors.length) throw new Error(checkoutCreate.checkoutUserErrors[ 0 ].message)

        window.location = checkoutCreate.checkout.webUrl
      } catch (error) {
        this.isLoading = false
        console.error(error)
        this.$notify({
          title: 'Whoops...',
          type: 'danger',
          message: 'Something went wrong - please try again.'
        })
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.input {
  height: 50px !important;
}
</style>
