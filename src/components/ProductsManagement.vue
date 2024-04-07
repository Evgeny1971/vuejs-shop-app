<template>
  <div>
    <h2>Product Management</h2>
    
    <!-- Sorting buttons -->
    <div>
      <button @click="sortBy('code')">Sort by Code</button>
      <button @click="sortBy('name')">Sort by Name</button>
      <button @click="sortBy('description')">Sort by Description</button>
      <button @click="sortBy('startOfPrice')">Sort by Start Of Price</button>
    </div>

    <!-- Form for adding/editing product -->
    <form @submit.prevent="addOrUpdateProduct">
      <label>Code:</label>
      <input type="text" v-model="product.code" required>
      <label>Name:</label>
      <input type="text" v-model="product.name" required>
      <label>Description:</label>
      <textarea v-model="product.description" required></textarea>
      <label>Start Of Price:</label>
      <input type="datetime-local" v-model="product.startOfPrice" required>
      <label>Image:</label>
      <input type="file" @change="handleFileUpload">
      <button type="submit">{{ editingMode ? 'Update Product' : 'Add Product' }}</button>
    </form>

    <!-- List of products -->
    <ul v-if="products.length > 0">
      <li v-for="product in sortedProducts" :key="product.id">
        <div>
          <img :src="getImageUrl(product.image)" alt="Product Image" style="max-width: 100px; max-height: 100px;">
        </div>
        <p>Code: {{ product.code }}</p>
        <p>Name: {{ product.name }}</p>
        <p>Description: {{ product.description }}</p>
        <p>Start Of Price: {{ formatDateTime(product.startOfPrice) }}</p>
        <button @click="editProduct(product)">Edit</button>
        <button @click="deleteProduct(product.id)">Delete</button>
      </li>
    </ul>

    <!-- Display a message if there are no products -->
    <p v-else>No products available.</p>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  data() {
    return {
      products: [],
      product: {
        id: null,
        code: '',
        name: '',
        description: '',
        startOfPrice: '', // datetime in format YYYY-MM-DDTHH:MM
        image: null, // store the filename of the uploaded image
      },
      selectedProduct: null,
      editingMode: false,
      sortByField: null,
      sortDesc: false,
    };
  },
  created() {
    this.fetchProducts();
  },
  methods: {
    fetchProducts() {
      axios.get('/api/products')
        .then(response => {
          this.products = response.data;
        })
        .catch(error => {
          console.error('Error fetching products:', error);
        });
    },
    addOrUpdateProduct() {
      if (this.editingMode) {
        this.updateProduct();
      } else {
        this.addProduct();
      }
    },
    addProduct() {
      // Upload image first, then add product
      this.uploadImage().then(() => {
        axios.post('/api/products', this.product)
          .then(response => {
            this.products.push(response.data);
            this.resetForm();
          })
          .catch(error => {
            console.error('Error adding product:', error);
          });
      });
    },
    uploadImage() {
      const formData = new FormData();
      formData.append('image', this.product.image);
      return axios.post('/api/upload', formData, {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      }).then(response => {
        this.product.image = response.data.filename;
      }).catch(error => {
        console.error('Error uploading image:', error);
      });
    },
    editProduct(product) {
      this.selectedProduct = product;
      this.product.id = product.id;
      this.product.code = product.code;
      this.product.name = product.name;
      this.product.description = product.description;
      this.product.startOfPrice = product.startOfPrice;
      this.editingMode = true;
    },
    updateProduct() {
      // Upload image first, then update product
      this.uploadImage().then(() => {
        axios.put(`/api/products/${this.product.id}`, this.product)
          .then(() => {
            const index = this.products.findIndex(p => p.id === this.product.id);
            if (index !== -1) {
              this.products.splice(index, 1, this.product);
            }
            this.resetForm();
          })
          .catch(error => {
            console.error('Error updating product:', error);
          });
      });
    },
    deleteProduct(productId) {
      axios.delete(`/api/products/${productId}`)
        .then(() => {
          this.products = this.products.filter(product => product.id !== productId);
        })
        .catch(error => {
          console.error('Error deleting product:', error);
        });
    },
    handleFileUpload(event) {
      this.product.image = event.target.files[0];
    },
    resetForm() {
      this.selectedProduct = null;
      this.product.id = null;
      this.product.code = '';
      this.product.name = '';
      this.product.description = '';
      this.product.startOfPrice = '';
      this.product.image = null;
      this.editingMode = false;
    },
    getImageUrl(filename) {
      return `/api/images/${filename}`;
    },
    sortBy(field) {
      if (this.sortByField === field) {
        this.sortDesc = !this.sortDesc;
      } else {
        this.sortByField = field;
        this.sortDesc = false;
      }
    },
    formatDateTime(datetime) {
      const options = { year: 'numeric', month: 'short', day: 'numeric', hour: '2-digit', minute: '2-digit' };
      return new Date(datetime).toLocaleDateString('en-US', options);
    },
  },
  computed: {
    sortedProducts() {
      if (!this.sortByField) return this.products;

      const sorted = this.products.slice().sort((a, b) => {
        const aValue = a[this.sortByField];
        const bValue = b[this.sortByField];

        if (this.sortDesc) {
          if (aValue < bValue) return 1;
          if (aValue > bValue) return -1;
          return 0;
        } else {
          if (aValue < bValue) return -1;
          if (aValue > bValue) return 1;
          return 0;
        }
      });

      return sorted;
    },
  },
};
</script>
