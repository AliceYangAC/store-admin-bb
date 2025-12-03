<template>
  <div class="action-button">
    <button @click="saveProduct" class="button">Save Product</button>
  </div>
  <br/>
  <div v-if="showValidationErrors" class="error">
    <br/>
    <ul v-for="error in validationErrors" :key="error">
      <li>{{ error }}</li>
    </ul>
  </div>
  <div class="product-form">
    <table>
      <tr>
        <td><label for="product-name">Name</label></td>
        <td><input id="product-name" placeholder="Product Name" v-model="product.name" /></td>
      </tr>

      <tr>
        <td><label for="product-price">Price</label></td>
        <td><input id="product-price" placeholder="Product Price" v-model="product.price" type="number" step="0.01" /></td>
      </tr>

      <tr>
        <td><label for="product-category">Category</label></td>
        <td><input id="product-category" placeholder="Product Category" v-model="product.category" /></td>
      </tr>

      <tr>
        <td><label for="product-brand">Brand</label></td>
        <td><input id="product-brand" placeholder="Product Brand" v-model="product.brand" /></td>
      </tr>

      <tr>
        <td><label for="product-description">Description</label></td>
        <td>
          <textarea rows="8" id="product-description" placeholder="Product Description" v-model="product.description" />
          <input type="hidden" id="product-id" v-model="product.id" />
        </td>
      </tr>

      <tr>
        <td><label for="product-image">Image</label></td>
        <td>
          <input type="file" @change="uploadImage" accept="image/*" />
          
          <div class="image-preview" v-if="product.image && product.image !== '/placeholder.png'">
            <img :src="resolveImageUrl(product.image)" alt="Product Preview" />
          </div>
          <input type="hidden" v-model="product.image" />
        </td>
      </tr>
    </table>
  </div>
</template>

<script>
  const productServiceUrl = '/product/';
  
  export default {
    name: 'ProductForm',
    // Added resolveImageUrl to props
    props: ['products', 'resolveImageUrl'], 
    emits: ['addProductsToList','updateProductInList'],
    data() {
      return {
        product: {
          id: 0,
          name: '',
          image: '/placeholder.png',
          description: '',
          price: 0.00,
          category: '',
          brand: '' 
        },
        showValidationErrors: false,
        isUploading: false
      }
    },
    mounted() {
      if (this.$route.params.id) {
        const product = this.products.find(product => product.id == this.$route.params.id)
        if (product) {
            this.product = Object.assign({}, product);
        }
      }
    },
    computed: {
      validationErrors() {
        let errors = [];
        if (!this.product.name) errors.push('Please enter a name');
        if (!this.product.description) errors.push('Please enter a description');
        if (this.product.price <= 0) errors.push('Price must be greater than 0');
        if (!this.product.category) errors.push('Please enter a category');
        if (!this.product.brand) errors.push('Please enter a brand');
        return errors;
      }
    },
    methods: {
      // REMOVED: resolveImageUrl() is now handled via props
      
      async uploadImage(event) {
        const file = event.target.files[0];
        if (!file) return;

        this.isUploading = true;
        const formData = new FormData();
        formData.append('file', file);

        try {
            const response = await fetch(`${productServiceUrl}upload`, {
                method: 'POST',
                body: formData
            });
            
            if (response.ok) {
                const data = await response.json();
                this.product.image = data.image; 
            } else {
                alert('Failed to upload image');
            }
        } catch (error) {
            console.error(error);
            alert('Error uploading image');
        } finally {
            this.isUploading = false;
        }
      },
      saveProduct() {
        if (this.validationErrors.length > 0) {
          this.showValidationErrors = true;
          return;
        }

        let method = 'PUT';
        let path = this.$route.path;
        if (path.includes('add')) {
          method = 'POST';
        }

        this.product.price = parseFloat(this.product.price);

        fetch(`${productServiceUrl}`, {
          method: method,
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify(this.product)
        })
          .then(response => response.json())
          .then(product => {
            alert('Product saved successfully');           
            if (method === 'PUT') {
              this.$emit('updateProductInList', this.product);
            } else {
              this.$emit('addProductsToList', product);
            }
            this.$router.push(`/product/${product.id}`);
          })
          .catch(error => {
            console.log(error)
            alert('Error occurred while saving product')
          })
      }
    }
  }
</script>

<style scoped>
ul {
  justify-content: center;
  list-style: none;
  margin: 0;
  padding: 0;
  width: 100%;
  color: #ff0000;
}

table {
  border-collapse: collapse;
  width: 100%;
  max-width: 600px;
}

td {
  padding: 10px;
  vertical-align: top;
}

label {
    font-weight: bold;
    display: block;
    margin-top: 5px;
}

.product-form {
  display: flex;
  justify-content: center;
}

.product-form input, 
.product-form textarea {
  padding: 8px;
  width: 100%;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.image-preview {
    margin-top: 10px;
    max-width: 200px;
    border: 1px solid #ddd;
    padding: 5px;
}

.image-preview img {
    width: 100%;
    height: auto;
}
</style>