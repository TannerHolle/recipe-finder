<template>
  <v-app>
    <v-app-bar
      app
      color="primary"
      dark
    >
      <div class="d-flex align-center">
        <v-img
          alt="Vuetify Logo"
          class="shrink mr-2"
          contain
          src="./assets/recipe-icon.png"
          transition="scale-transition"
          width="40"
        />
        Recipe Finder
      </div>

      <v-spacer></v-spacer>

      <span class="mr-2">Tanner Holle</span>
    </v-app-bar>

    <v-main>
      <h2>Recipe Finder</h2>
      <v-container fluid>
        <v-row justify="center" align="center" mb-4>
          <v-col cols="12" md="6">
            <v-text-field
              v-model="ingredientInput"
              label="Enter ingredients (comma-separated)"
              class="text-center"
            ></v-text-field>
            <!-- Wrapper row for radio buttons and dropdown -->
            <v-row>
              <!-- Radio buttons -->
              <v-col>
                <v-radio-group v-model="searchType">
                  <v-radio label="Include Ingredients" value="includeIngredients"></v-radio>
                  <v-radio label="Only Ingredients" value="onlyIngredients"></v-radio>
                </v-radio-group>
              </v-col>

              <v-col class="ml-auto">
                <!-- Input field for max time -->
                <v-text-field
                  v-model="maxTime"
                  label="Max Time (in minutes)"
                  type="number"
                  class="text-center"
                ></v-text-field>
              </v-col>

              <!-- Dropdown for difficulty level -->
              <v-col class="ml-auto">
                <v-select
                  v-model="difficultyLevel"
                  :items="['Easy', 'Intermediate', 'Advanced']"
                  label="Difficulty"
                  multiple
                ></v-select>
              </v-col>

                <!-- Dropdown for dish type -->
              <v-col class="ml-auto">
                <v-select
                  v-model="dishType"
                  :items="['Main Dish', 'Side Dish', 'Dessert', 'Appetizers']"
                  label="Dish Type"
                  multiple
                ></v-select>
              </v-col>
            </v-row>

            <!-- Single button for search -->
            <v-btn @click="search" color="primary">Search Recipes</v-btn>
          </v-col>
        </v-row>
        <v-row justify="center" align="center" mb-4>
          <v-col cols="12" md="10">
            <v-expansion-panels v-if="!isLoading">
              <v-expansion-panel
              v-for="(recipe, index) in paginatedRecipes" :key="index" @click="toggleIngredients(index)"
              >
              <v-expansion-panel-header>
                <h3>{{ recipe.title }}</h3>
              </v-expansion-panel-header>
              <v-expansion-panel-content class="text-left">
                <br>
                <h4>Ingredients:</h4>
                <ul>
                  <li v-for="(ingredient, idx) in ingredients" :key="idx">{{ ingredient }}</li>
                </ul>
                <br><br>
                <h4>Directions:</h4>
                <ol>
                  <li v-for="(direction, idx) in directions" :key="idx">{{ direction }}</li>
                </ol>
                <br>
                <h4>Level:</h4> {{ recipe.level }}
                <h4>Time to Make:</h4> {{ recipe.time }}
                <h4>Category:</h4> {{ recipe.category }}
                {{ recipe.note }}
                {{ recipe.author }}
              </v-expansion-panel-content>
            </v-expansion-panel>
            <div v-if="recipes.length > itemsPerPage" class="text-center mt-4">
              <v-btn @click="changePage('prev')" :disabled="currentPage === 1">Previous Page</v-btn>

              <!-- Always display the first page -->
              <v-chip @click="goToPage(1)" :color="currentPage === 1 ? 'primary' : ''" class="mx-1">
                1
              </v-chip>

              <!-- Display ellipsis if there are more pages before the current page -->
              <v-chip v-if="currentPage > 4">...</v-chip>

              <!-- Display pages around the current page -->
              <span v-for="page in visiblePages" :key="page">
                <v-chip
                  v-if="page !== totalPages"
                  @click="goToPage(page)"
                  :color="currentPage === page ? 'primary' : ''"
                  class="mx-1"
                >
                  {{ page }}
                </v-chip>
              </span>

              <!-- Display ellipsis if there are more pages after the current page -->
              <v-chip v-if="currentPage < totalPages - 3">...</v-chip>

              <!-- Always display the last page -->
              <v-chip @click="goToPage(totalPages)" :color="currentPage === totalPages ? 'primary' : ''" class="mx-1">
                {{ totalPages }}
              </v-chip>

              <v-btn @click="changePage('next')" :disabled="currentPage === totalPages">Next Page</v-btn>
            </div>
          </v-expansion-panels>
          <div class="loading-spinner" v-if="isLoading"></div>
          </v-col>
        </v-row>
      </v-container>
    </v-main>
  </v-app>
</template>

<script>

export default {
  data() {
    return {
      ingredientInput: '',
      recipes: [],
      ingredients: [],
      directions: [],
      matchedRecipes: [],
      isLoading: false,
      searchType: 'includeIngredients',
      difficultyLevel: '',
      dishType: '',
      maxTime: '', 
      // Pagination properties
      itemsPerPage: 15, // Adjust as needed
      currentPage: 1,
    };
  },
  computed: {
    paginatedRecipes() {
      const startIndex = (this.currentPage - 1) * this.itemsPerPage;
      const endIndex = startIndex + this.itemsPerPage;
      return this.recipes.slice(startIndex, endIndex);
    },
    totalPages() {
      return Math.ceil(this.recipes.length / this.itemsPerPage);
    },
    visiblePages() {
      const maxVisiblePages = 5; // Adjust as needed
      const startPage = Math.max(2, Math.min(this.currentPage - Math.floor(maxVisiblePages / 2), this.totalPages - maxVisiblePages + 1));

      return Array.from({ length: Math.min(this.totalPages - 1, maxVisiblePages) }, (_, index) => startPage + index);
    },
  },
  methods: {
    async searchRecipes() {
      this.isLoading = true;
      this.currentPage = 1
      await this.pause(50)
      const recipes = require('./assets/recipes-v4.json');
      const ingredientsProvided = this.ingredientInput.split(',').map(ingredient => ingredient.trim());

      // Define a dictionary of excluded ingredients and their variations
      const excludedIngredients = {
        'chicken': ['stock', 'broth', 'flavored', 'pan drippings'],
        'beef': ['stock', 'broth'],
        'butter': ['peanut'],
        'sugar': ['brown'],
        'cheese': ['cream', 'cottage'],
        // Add other variations as needed
      };

      // Function to check if a recipe contains any allowed variation for a specific ingredient
      const containsAllowedVariation = (recipe, ingredient) =>
        recipe.ingredients.some(item =>
          new RegExp(`\\b${ingredient}\\b`, 'i').test(item) &&
          !excludedIngredients[ingredient].some(variation => new RegExp(`\\b${variation}\\b`, 'i').test(item))
        );

      // Filter out recipes containing excluded variations for each ingredient
      const recipesWithValidIngredients = recipes.filter(recipe =>
        ingredientsProvided.every(ingredient =>
          !excludedIngredients[ingredient] || containsAllowedVariation(recipe, ingredient)
        )
      );

      // Continue with your existing logic for filtering based on provided ingredients, difficulty level, and dish type
      const containsAllIngredients = (recipe, ingredients) =>
        ingredients.every(ingredient => recipe.ingredients.some(item => new RegExp(`\\b${ingredient}\\b`, 'i').test(item)));

      const validRecipes = recipesWithValidIngredients.filter(recipe =>
        containsAllIngredients(recipe, ingredientsProvided)
      );


      // Additional filtering based on difficulty level
      const difficultyFilteredRecipes = this.difficultyLevel && this.difficultyLevel.length
        ? validRecipes.filter(recipe => {
            const selectedDifficulties = this.difficultyLevel.map(difficulty => difficulty.toLowerCase());
            return selectedDifficulties.includes(recipe.level && recipe.level.toLowerCase());
          })
        : validRecipes;

      // Additional filtering based on dish type (category)
      const dishTypeFilteredRecipes = this.dishType && this.dishType.length
        ? difficultyFilteredRecipes.filter(recipe => {
            const selectedDishTypes = this.dishType.map(type => type.toLowerCase());
            return selectedDishTypes.includes(recipe.category && recipe.category.toLowerCase());
          })
        : difficultyFilteredRecipes;

      // Filter recipes based on maxTime
      const maxTimeInMinutes = this.maxTime ? parseInt(this.maxTime, 10) : Infinity;
      const timeFilteredRecipes = dishTypeFilteredRecipes.filter(recipe =>
        recipe.time.trim() !== '' && this.convertTimeToMinutes(recipe.time) <= maxTimeInMinutes
      );

      console.log(this.convertTimeToMinutes("32 min"))

      this.recipes = timeFilteredRecipes.map(recipe => ({ ...recipe, showIngredients: false }));
      this.isLoading = false;
    }, 
    async findRecipes() {
      this.isLoading = true;
      await this.pause(50)
      const recipes = require('./assets/recipes-v4.json');
      const basicSpices = ['salt', 'pepper', 'garlic powder', 'onion powder', 'paprika', 'cayenne', 'thyme']
      const ingredientsProvided = this.ingredientInput.split(',').map(ingredient => ingredient.trim());
      const allIngredients = basicSpices.concat(ingredientsProvided)
      const containsAllIngredients = (recipe, ingredients) => recipe.ingredients.every(ingredient => ingredients.some(item => new RegExp(`\\b${item}\\b`, 'i').test(ingredient)));
      const recipesWithIngredients = recipes.filter(recipe => containsAllIngredients(recipe, allIngredients));

      // Additional filtering based on difficulty level
      const difficultyFilteredRecipes = this.difficultyLevel && this.difficultyLevel.length
        ? recipesWithIngredients.filter(recipe => {
            const selectedDifficulties = this.difficultyLevel.map(difficulty => difficulty.toLowerCase());
            return selectedDifficulties.includes(recipe.level && recipe.level.toLowerCase());
          })
        : recipesWithIngredients;

      // Additional filtering based on dish type (category)
      const dishTypeFilteredRecipes = this.dishType && this.dishType.length
        ? difficultyFilteredRecipes.filter(recipe => {
            const selectedDishTypes = this.dishType.map(type => type.toLowerCase());
            return selectedDishTypes.includes(recipe.category && recipe.category.toLowerCase());
          })
        : difficultyFilteredRecipes;

      this.recipes = dishTypeFilteredRecipes.map(recipe => ({ ...recipe, showIngredients: false }));
      this.isLoading = false;
    }, 
    search() {
      if (this.searchType === 'includeIngredients') {
        this.searchRecipes();
      } else if (this.searchType === 'onlyIngredients') {
        this.findRecipes();
      }
    },
    convertTimeToMinutes(time) {
      const timeRegex = /(?:([\d]+)\s*day)?\s*(?:([\d]+)\s*hr)?\s*(?:([\d]+)\s*min)?/i;
      const match = time.match(timeRegex);
      if (match) {
        const days = match[1] ? parseInt(match[1], 10) : 0;
        const hours = match[2] ? parseInt(match[2], 10) : 0;
        const minutes = match[3] ? parseInt(match[3], 10) : 0;
        return days * 24 * 60 + hours * 60 + minutes;
      }
      return 0;
    },
    toggleIngredients(index) {
      this.ingredients = this.recipes[index].ingredients
      this.directions = this.recipes[index].directions
      this.recipes[index].showIngredients = !this.recipes[index].showIngredients;
    },
    changePage(direction) {
      if (direction === 'prev' && this.currentPage > 1) {
        this.currentPage--;
      } else if (direction === 'next' && this.currentPage < this.totalPages) {
        this.currentPage++;
      }
    },
    goToPage(page) {
      if (page >= 1 && page <= this.totalPages) {
        this.currentPage = page;
      }
    },
    getVisiblePages() {
      const visiblePages = [];
      const maxVisiblePages = 5; // Adjust as needed

      // Add current page
      visiblePages.push(this.currentPage);

      // Add pages before the current page
      for (let i = this.currentPage - 1; i >= Math.max(1, this.currentPage - maxVisiblePages); i--) {
        visiblePages.unshift(i);
      }

      // Add pages after the current page
      for (let i = this.currentPage + 1; i <= Math.min(this.totalPages, this.currentPage + maxVisiblePages); i++) {
        visiblePages.push(i);
      }

      return visiblePages;
    },
    handleEnterKey(event) {
      // Check if the pressed key is 'Enter' and trigger the search
      if (event.key === 'Enter') {
        this.search();
      }
    },
    pause(duration) {
      return new Promise(resolve => {
        setTimeout(() => {
          resolve();
        }, duration);
      });
    }
  },
  mounted() {
    // Listen for the 'enter' key press to trigger the search
    window.addEventListener('keydown', this.handleEnterKey);
  },
  beforeDestroy() {
    // Remove the event listener when the component is destroyed
    window.removeEventListener('keydown', this.handleEnterKey);
  },
};
</script>

<style scoped>
  .loading-spinner {
    /* Add your styling for the loading spinner */
    /* For example: display a spinning animation */
    border: 4px solid rgba(0, 0, 0, 0.1);
    border-top: 4px solid #3498db;
    border-radius: 50%;
    width: 40px;
    height: 40px;
    animation: spin 1s linear infinite;
    margin: 20px auto;
  }

  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  
  #app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
  }


</style>
