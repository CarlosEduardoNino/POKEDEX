<template>
  <div class="body">
  

   
  <div class="pokedex">
    
    <div class="pokedex-screen">

      <div class="button-container">
  <button class="round-button" :class="{ active: isActive }"></button>
  <button class="round-button" :class="{ active: isActive }"></button>
  <button class="round-button" :class="{ active: isActive }"></button>
  
</div>

      <h2 v-if="!pokemonData">Bienvenido a la Pokédex</h2>
      <div v-if="errorOccurred" class="error">
  <img 
    src="https://i.pinimg.com/564x/4c/10/03/4c1003e52e558e384f99ef4d19dc5c47.jpg" 
    alt="Error" 
    class="error-image" 
  />
  <p>Ocurrió un error al cargar el Pokémon.</p>
</div>


  <div v-if="pokemonData" class="pokemon-info">
    <h2>{{ capitalize(pokemonData.name) }} no. {{ pokemonData.id }}</h2>
    <div :class="['main-screen', getTypeClass()]">
      <div class="pokemon-stats">
        <p>Peso: {{ pokemonData.weight / 10 }} kg</p>
        <p>Altura: {{ pokemonData.height / 10 }} m</p>
      </div>
      <img :src="pokemonData.sprites.front_default" :alt="pokemonData.name" class="pokemon-image">
    </div>
    <div class="stats-screen">
      <div v-for="stat in pokemonData.stats" :key="stat.stat.name" class="stat-bar">
        <p>{{ capitalize(stat.stat.name) }}</p>
        <div class="progress-bar">
          <div :style="{ width: getStatWidth(stat.base_stat) }" class="progress"></div>
          <span class="stat-value">{{ stat.base_stat }}</span>
        </div>
      </div>
    </div>
  </div>
  <div v-else class="welcome-screen">
    <p>Ingresa el nombre de un Pokémon para comenzar</p>
   
  </div>
  
  
</div>

    <div class="pokedex-info">
      <div class="description">
        <p>{{ pokemonDescription }}</p>
      </div>
      <div class="types">
        <h3>TYPES</h3>
        <button
          v-for="type in pokemonData?.types"
          :key="type.type.name"
          :class="['type-button', type.type.name]"
        >
          {{ type.type.name.toUpperCase() }}
        </button>
      </div>
      <div class="weaknesses">
        <h3>WEAKNESSES</h3>
        <div class="weakness-grid">
          <span
            v-for="weakness in weaknesses"
            :key="weakness"
            :class="['weakness-badge', weakness]"
          >
            {{ weakness.toUpperCase() }}
          </span>
        </div>
      </div>
      <p>Evolution</p>
      <div class="evolution-chain">
        <br>
        <div v-for="evolution in evolutionChain" :key="evolution.name" class="evolution-item">
          <img :src="evolution.sprite" :alt="evolution.name" />
          <p>{{ capitalize(evolution.name) }}</p>
        </div>
      </div>
      <div class="controls">
        <input v-model="pokemonName" type="text" placeholder="Nombre del Pokémon" class="pokemon-input" />
        <button @click="fetchPokemon" class="search-button">SEARCH</button>

        <div class="nav-buttons">
          <button @click="previousPokemon" class="nav-button">◀</button>
          <span>{{ pokemonData ? pokemonData.id : 0 }}</span>
          <button @click="nextPokemon" class="nav-button">▶</button>
        </div>
      </div>
    </div>
  </div>
</div>
</template> 

<script >
import { ref } from 'vue';
import axios from 'axios';



export default {
  setup() {
    const pokemonName = ref('');
    const pokemonData = ref(null);
    const pokemonDescription = ref('');
    const evolutionChain = ref([]);
    const weaknesses = ref([]);
    const errorOccurred = ref("");
    const isActive = ref(false);

    const fetchPokemon = async () => {
  isActive.value = true; 
  errorOccurred.value = false; 

  
  const query = pokemonName.value.trim().toLowerCase();
  if (!query) {
    pokemonDescription.value = 'Por favor, ingresa el nombre de un Pokémon.';
    isActive.value = false;
    return;
  }

  try {
    const response = await axios.get(`https://pokeapi.co/api/v2/pokemon/${query}`);
    pokemonData.value = response.data;
    await fetchPokemonDescription(response.data.id);
    await fetchEvolutionChain(response.data.id);
    await fetchWeaknesses(response.data.types);
  } catch (err) {
    console.error('Error al obtener el Pokémon:', err);
    pokemonData.value = null;
    pokemonDescription.value = 'Pokémon no encontrado. Verifica el nombre o ID e intenta de nuevo.';
    evolutionChain.value = [];
    weaknesses.value = [];
    errorOccurred.value = true; 
  } finally {
    isActive.value = false; 
  }
};


    const fetchPokemonDescription = async (id) => {
      try {
        const response = await axios.get(`https://pokeapi.co/api/v2/pokemon-species/${id}`);
        const englishEntry = response.data.flavor_text_entries.find(entry => entry.language.name === 'en');
        pokemonDescription.value = englishEntry ? englishEntry.flavor_text.replace(/\f/g, ' ') : 'Descripción no disponible.';
      } catch (err) {
        console.error('Error al obtener la descripción del Pokémon:', err);
        pokemonDescription.value = 'Descripción no disponible.';
      }
    };

    const fetchEvolutionChain = async (id) => {
      try {
        const speciesResponse = await axios.get(`https://pokeapi.co/api/v2/pokemon-species/${id}`);
        const evolutionChainResponse = await axios.get(speciesResponse.data.evolution_chain.url);
        const chain = evolutionChainResponse.data.chain;

        evolutionChain.value = await getEvolutionChainData(chain);
      } catch (err) {
        console.error('Error al obtener la cadena de evolución:', err);
        evolutionChain.value = [];
      }
    };

    const getEvolutionChainData = async (chain) => {
      const evolution = [];
      let currentPokemon = chain;

      while (currentPokemon) {
        try {
          const pokemonData = await axios.get(`https://pokeapi.co/api/v2/pokemon/${currentPokemon.species.name}`);
          evolution.push({
            name: currentPokemon.species.name,
            sprite: pokemonData.data.sprites.front_default,
          });

          currentPokemon = currentPokemon.evolves_to[0];
        } catch (error) {
          console.error('Error al obtener los datos de evolución:', error);
          break;
        }
      }

      return evolution;
    };

    const fetchWeaknesses = async (types) => {
      try {
        const typeData = await Promise.all(
          types.map(type => axios.get(type.type.url))
        );

        const damageRelations = typeData.map(data => data.data.damage_relations.double_damage_from);
        const allWeaknesses = new Set();

        damageRelations.forEach(relations => {
          relations.forEach(relation => allWeaknesses.add(relation.name));
        });

        weaknesses.value = Array.from(allWeaknesses);
      } catch (err) {
        console.error('Error al obtener las debilidades:', err);
        weaknesses.value = [];
      }
    };
    
    const capitalize = (str) => str.charAt(0).toUpperCase() + str.slice(1);

    const previousPokemon = () => {
  if (pokemonData.value && pokemonData.value.id > 1) {
    pokemonName.value = (pokemonData.value.id - 1).toString();
    fetchPokemon();
  }
};

const nextPokemon = () => {
  if (pokemonData.value && pokemonData.value.id < 300) {
    pokemonName.value = (pokemonData.value.id + 1).toString();
    fetchPokemon();
  }
};

    const getStatWidth = (baseStat) => {
      const maxStat = 255;
      return `${(baseStat / maxStat) * 100}%`;
    };

    const getTypeClass = () => {
      
      if (pokemonData.value && pokemonData.value.types.length > 0) {
        return pokemonData.value.types[0].type.name;
      }
      return '';
    };

    return {
      pokemonName,
      pokemonData,
      pokemonDescription,
      evolutionChain,
      weaknesses,
      fetchPokemon,
      capitalize,
      previousPokemon,
      nextPokemon,
      getStatWidth,
      getTypeClass,
      errorOccurred, 
      isActive,
      fetchPokemon,
    };
  },
};
</script>


<style scoped>
@import url('https://fonts.googleapis.com/css2?family=VT323&display=swap');
*{
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: 'VT323', monospace;
}
.body {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center; 
  height: 100vh; 
  background-image: url("https://i.pinimg.com/564x/59/2c/8c/592c8c67092d24e7ca8c728044c9b1c2.jpg");
  background-position: left 320px;

 
}


.progress {
  height: 100%;
  background-color: #53f758;
  transition: width 0.3s;
  border-radius: 3px 0 0 3px;
  box-shadow: inset 0 0 5px rgba(0, 0, 0, 0.3);
  position: relative; 
}

.stat-value {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color:black;
  font-weight: bold;
  font-size: 24px;
}


.pokedex {
  font-family: 'VT323', monospace;
  display: flex;
  background-color: #c21807;
  border-radius: 10px;
  padding: 20px;

  max-width: 1400px;
  margin: 0 auto;
}
.error {
  text-align: center; 
  margin-top: 20px; 
  
  
}
.error img {
width: 250px
;
  height: 250px; 
  display: block; 
  margin: 0 auto; 
}

.button-container {
  display: flex;
  gap: 10px;
  margin-bottom: 20px; 
}

.round-button {
  width: 35px;
  height: 35px;
  border: none;
  border-radius: 50%;
  background-color: #525bcf; 
  transition: background-color 0.5s ease;
  cursor: pointer;
}

.round-button.active {
  background-color: #ffd001; 
}

.nav-button {
  background-color: #333;
  color: white;
  border: 2px solid #ffcb05; 
  padding: 10px 15px; 
  border-radius: 8px; 
  cursor: pointer;
  font-family: inherit;
  font-size: 24px;
  transition: transform 0.2s, background-color 0.2s;
}

.nav-button:hover {
  background-color: #ffcb05; 
  color: #333;
  transform: scale(1.1); 
}
.pokedex-screen h2 {
  font-size: 36px; 
  color: #ffcb05; 
  text-align: center; 
  margin-bottom: 20px; 
} 


.pokedex-screen {
  background-color: #dfebdf;
  border: 10px solid #333;
  border-radius: 10px;
  padding: 20px;
  width: 50%;
  height: 670px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
}

.main-screen {
  background-color: #ffffff;
  border: 5px solid #333;
  margin-bottom: 10px;
  height: 200px;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  border-radius: 70px 15px 15px 15px;
  
}
.pokedex-screen {
  background-color:white;
  border: 10px solid #333;
  border-radius: 10px;
  padding: 20px;
  width: 50%;
  height: 680px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;


}

.pokemon-stats {
  margin-top: 10px;
  text-align: center;
  font-size: 20px;
  color: #333;
}


.pokemon-image {
  width: 250px;
  height: 250px;
  image-rendering: pixelated;
}

.pokemon-info h2 {
  font-size: 40px;
}


.stat-bar {
  display: flex;
  align-items: center;
  margin-bottom: 25px;
}

.stat-bar p {
  width: 120px;
  margin: 0;
  font-size: 25px;
  color: #333;
}

.progress-bar {
  background-color: #ddd;
  border: 2px solid #333;
  border-radius: 5px;
  width: 100%;
  height: 20px;
  overflow: hidden;
  margin-left: 10px;
  position: relative;
}

.progress {
  height: 100%;
  background-color: #4caf50;
  transition: width 0.3s;
  border-radius: 3px 0 0 3px;
  box-shadow: inset 0 0 5px rgba(0, 0, 0, 0.3);
}


.description {
  background-color: #e6ffe6; 
  border: 2px solid #333;
  padding: 10px;
  margin-top: 10px;
  border-radius: 8px;
  height: 100px;
  overflow-y: auto;
}

.description p {
  font-size: 25px;
}


.pokedex-info {
  width: 50%;
  padding-left: 20px;
}


.types {
  margin-bottom: 20px;
}

.types h3 {
  font-size: 24px;
}

.type-button {
  padding: 5px 10px;
  margin-right: 5px;
  border: none;
  border-radius: 5px;
  color: white;
  font-family: inherit;
  cursor: pointer;
  font-size: 24px;
}


.weaknesses {
  margin-bottom: 20px;
}

.weaknesses h3 {
  font-size: 24px;
  margin-bottom: 10px;
}

.weakness-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
  gap: 10px;
}

.weakness-badge {
  padding: 10px;
  border-radius: 5px;
  color: white;
  font-family: inherit;
  text-align: center;
  display: inline-block;
}


.evolution-chain {
  display: flex;
  justify-content: space-between;
  margin-bottom: 20px;
}

.evolution-item {
  text-align: center;
}

.evolution-item img {
  width: 140px;
  height: 120px;
}

.evolution-item p {
  font-size: 22px;
}


.controls {
  display: flex;
  flex-direction: column;
}

.pokemon-input {
  margin-bottom: 10px;
  padding: 5px;
  font-family: inherit;
  font-size: 24px;
}

.search-button {
  background-color: #ffcb05;
  border: none;
  padding: 10px;
  font-family: inherit;
  cursor: pointer;
  margin-bottom: 10px;
  font-size: 24px;
}

.nav-buttons {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.nav-button {
  background-color: #333;
  color: white;
  border: none;
  padding: 5px 10px;
  font-family: inherit;
  cursor: pointer;
  font-size: 24px;
}


.main-screen {
  border: 5px solid #333;
  margin-bottom: 10px;
  height: 200px;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  border-radius: 70px 15px 15px 15px;
  transition: background-color 0.3s;
}
.pokedex-title {
  font-size: 36px; 
  color: #ffcb05;
  text-align: center; 
  margin-bottom: 20px; 
}


.normal { background-color: #A8A878; }
.fire { background-color: #F08030; }
.water { background-color: #6890F0; }
.electric { background-color: #F8D030; }
.grass { background-color: #78C850; }
.ice { background-color: #98D8D8; }
.fighting { background-color: #C03028; }
.poison { background-color: #A040A0; }
.ground { background-color: #E0C068; }
.flying { background-color: #A890F0; }
.psychic { background-color: #F85888; }
.bug { background-color: #A8B820; }
.rock { background-color: #B8A038; }
.ghost { background-color: #705898; }
.dragon { background-color: #7038F8; }
.dark { background-color: #705848; }
.steel { background-color: #B8B8D0; }
.fairy { background-color: #EE99AC; }

p {
  font-size: 30px; 
  color: rgb(255, 205, 5);
  text-shadow: 2px 2px 2px rgb(255, 205, 5); 
  text-transform: uppercase; 
  font-family: inherit;
  margin-top: 10px;
}


.types h3 {
  font-size: 28px; 
  color: rgb(255, 205, 5);
  text-shadow: 2px 2px 2px rgb(255, 205, 5); 
  text-transform: uppercase; 
  margin-bottom: 15px;
}

.type-button {
  font-size: 22px; 
  border: 2px solid #333; 
  text-shadow: 1px 1px 1px rgba(0, 0, 0, 0.3);
  padding: 10px 15px; 
  border-radius: 10px; 
  transition: background-color 0.3s, transform 0.3s; 
}

.type-button:hover {
  background-color: #ffcb05;
  color: #333;
  transform: scale(1.1); 
}

.weaknesses h3 {
  font-size: 28px;
  color:rgb(255, 205, 5);
  text-shadow: 2px 2px 2px rgb(255, 206, 8);
  text-transform: uppercase;
  margin-bottom: 15px;
}

.weakness-badge {
  font-size: 20px; 
  border: 2px solid #333; 
  padding: 8px 12px;
  text-shadow: 1px 1px 1px rgba(0, 0, 0, 0.3); 
  background-color: rgba(255, 203, 5, 0.7); 
  transition: transform 0.3s; 
}

.weakness-badge:hover {
  transform: scale(1.1); 
}

.evolution-chain {
  display: flex;
  justify-content: space-between;
  margin-bottom: 20px;
}

.evolution-item p {
  font-size: 22px;
  font-weight: bold;
  color: #333;
  text-shadow: 2px 2px 2px rgba(0, 0, 0, 0.2);
  margin-top: 10px;
}

.evolution-item img {
  border: 2px solid #333; 
  border-radius: 8px; 
  box-shadow: 3px 3px 5px rgba(0, 0, 0, 0.3); 
  transition: transform 0.3s;
}

.evolution-item img:hover {
  transform: scale(1.05); 
}





.controls input, 
.controls button, 
.controls span {
  font-size: 24px; 
}

@media (max-width: 900px) {
  .body {
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center; 
  
  }

  
.pokemon-image {
  width: 150px;
  height: 150px;
  image-rendering: pixelated;
}

  .pokedex {
    flex-direction: column; 
 
  }
.evolution-item img {
  width: 110px;
  height: 90px;
}
  .pokedex-screen,
  .pokedex-info {
    width:100%; 
    
    padding: 10px; 
  }
  .pokedex-info{
order:1;
  }
  .pokedex-screen{
    order:2;
  }
  .pokedex-info {
    margin-top: 20px; 
  }
}
@media (max-width: 700px) {
  .pokedex {
    flex-direction: column; 
  }

  .pokedex-screen,
  .pokedex-info {
    width: 95%; 
    padding: 10px; 
  }

  .pokedex-info {
    margin-top: 20px; 
  }
}
@media (max-width: 440px){
  .pokedex-screen {
    height: 100%;
  }
    .body{
      height: auto;
      width: 100%;
      flex-direction: column;
    }
}
</style>