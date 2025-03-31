<template>
    <div class="container">
      <h2>Condução em Placa Plana</h2>
      
      <div class="input-section">
        <h3>Parâmetros Gerais</h3>
        <div class="input-group">
          <label>
            Posição Inicial (x₁) [m]:
            <input type="number" v-model.number="x1" step="0.01">
          </label>
          <label>
            Posição Final (x₂) [m]:
            <input type="number" v-model.number="x2" step="0.01">
          </label>
          <label>
            Área [m²]:
            <input type="number" v-model.number="L" step="0.01">
          </label>
          <label>
            Condutividade Térmica (k) [W/m·K]:
            <input type="number" v-model.number="k" step="1">
          </label>
        </div>
      </div>
  
      <div class="columns">
        <div class="input-section">
          <h3>Condição em x₁</h3>
          <select v-model="innerBCType">
            <option value="temp">Temperatura</option>
            <option value="flux">Fluxo Térmico</option>
            <option value="conv">Convecção</option>
          </select>
          <div class="input-group">
            <div v-if="innerBCType === 'temp'">
              <label>Temperatura (T₁) [°C]:</label>
              <input type="number" v-model.number="innerTemp">
            </div>
            <div v-else-if="innerBCType === 'flux'">
              <label>Fluxo Térmico (q₁) [W/m²]:</label>
              <input type="number" v-model.number="innerFlux">
            </div>
            <div v-else-if="innerBCType === 'conv'">
              <label>Coef. Convecção (h₁) [W/m²·K]:</label>
              <input type="number" v-model.number="innerH">
              <label>Temperatura Ambiente (T∞₁) [°C]:</label>
              <input type="number" v-model.number="innerTInf">
            </div>
          </div>
        </div>
  
        <div class="input-section">
          <h3>Condição em x₂</h3>
          <select v-model="outerBCType">
            <option value="temp">Temperatura</option>
            <option value="flux">Fluxo Térmico</option>
            <option value="conv">Convecção</option>
          </select>
          <div class="input-group">
            <div v-if="outerBCType === 'temp'">
              <label>Temperatura (T₂) [°C]:</label>
              <input type="number" v-model.number="outerTemp">
            </div>
            <div v-else-if="outerBCType === 'flux'">
              <label>Fluxo Térmico (q₂) [W/m²]:</label>
              <input type="number" v-model.number="outerFlux">
            </div>
            <div v-else-if="outerBCType === 'conv'">
              <label>Coef. Convecção (h₂) [W/m²·K]:</label>
              <input type="number" v-model.number="outerH">
              <label>Temperatura Ambiente (T∞₂) [°C]:</label>
              <input type="number" v-model.number="outerTInf">
            </div>
          </div>
        </div>
  
        <div class="input-section">
          <h3>Ponto de Análise</h3>
          <div class="input-group">
            <label>
              Posição (x) [m]:
              <input type="number" v-model.number="xInput" step="0.01">
            </label>
          </div>
        </div>
      </div>
  
      <div class="error" v-if="errors.length">
        <div v-for="error in errors" :key="error">{{ error }}</div>
      </div>
  
      <div class="results">
        <h3>Resultados</h3>
        <div v-if="Q !== null">
          <p>Taxa de Transferência de Calor (Q): {{ Q.toFixed(4) }} W</p>
          <p v-if="Tx !== null">Temperatura em (x) = {{ xInput }} m: {{ Tx.toFixed(4) }} °C</p>
        </div>
      </div>
  
      <div id="plot"></div>
    </div>
</template>  
  
<script>
import { ref, watch, onMounted } from 'vue';
import Plotly from 'plotly.js-dist-min';
  
// Definiçcao das condicoes de contorno para placa plana
const boundaryConditions = {
  temp: (x, T) => ({ a: x, b: 1, c: T }),
  flux: (x, q, k) => ({ a: 1, b: 0, c: -q / k }),
  conv: (x, h, Tinf, k) => {
    if (x === 0) {
      return { a: k, b: -h, c: -h * Tinf };
    } else {
      return { a: h * x + k, b: h, c: h * Tinf };
    }
  }
};

export default {
    name: 'PlateConduction',
    setup() {
      // Parametros gerais: x variando de x1 a x2 (de um lado a outro da placa)
      const x1 = ref(null);
      const x2 = ref(null);
      const L = ref(null); // Área 'inputada' pelo usuario
      const k = ref(null);
      
      // Condição na face em x1 (seria tipo inicial)
      const innerBCType = ref('temp');
      const innerTemp = ref(null);
      const innerFlux = ref(null);
      const innerH = ref(null);
      const innerTInf = ref(null);
      
      // Condição na face em x2 (seria tipo final)
      const outerBCType = ref('temp');
      const outerTemp = ref(null);
      const outerFlux = ref(null);
      const outerH = ref(null);
      const outerTInf = ref(null);
      
      // Ponto de analise da temperatura (solicitada pela questao)
      const xInput = ref(null);
      
      // Resultados
      const Q = ref(null);
      const Tx = ref(null);
      
      const errors = ref([]);
  
      // Validacoes dos inputs
      const validateInputs = () => {
        errors.value = [];
        if (x1.value === null || x2.value === null || x1.value >= x2.value) {
          errors.value.push("É obrigatório que x₁ < x₂.");
        }
        if (k.value === null || k.value <= 0) {
          errors.value.push("Condutividade térmica (k) deve ser positiva (k > 0).");
        }
        if (L.value === null || L.value <= 0) {
          errors.value.push("O valor de L (área) deve ser positivo.");
        }
        if (xInput.value === null || xInput.value < x1.value || xInput.value > x2.value) {
          errors.value.push("Valor de x para análise deve estar entre x₁ e x₂.");
        }
        
        // Validacao das CCs em x1
        if (innerBCType.value === 'temp' && innerTemp.value === null) {
          errors.value.push("Temperatura em x₁ é obrigatória.");
        }
        if (innerBCType.value === 'flux' && innerFlux.value === null) {
          errors.value.push("Fluxo em x₁ é obrigatório.");
        }
        if (innerBCType.value === 'conv') {
          if (innerH.value === null || innerH.value <= 0) {
            errors.value.push("h₁ inválido.");
          }
          if (innerTInf.value === null) {
            errors.value.push("Temperatura ambiente (T∞₁) é obrigatória.");
          }
        }
        
        // Validacao das CCs em x2
        if (outerBCType.value === 'temp' && outerTemp.value === null) {
          errors.value.push("Temperatura em x₂ é obrigatória.");
        }
        if (outerBCType.value === 'flux' && outerFlux.value === null) {
          errors.value.push("Fluxo em x₂ é obrigatório.");
        }
        if (outerBCType.value === 'conv') {
          if (outerH.value === null || outerH.value <= 0) {
            errors.value.push("h₂ inválido.");
          }
          if (outerTInf.value === null) {
            errors.value.push("Temperatura ambiente (T∞₂) é obrigatória.");
          }
        }
        
        // Se ambas as condicoes forem de fluxo, os fluxos devem ser iguais (fluxo constante na placa)
        if (innerBCType.value === 'flux' && outerBCType.value === 'flux') {
          if (innerFlux.value !== outerFlux.value) {
            errors.value.push("Em condições de fluxo especificado em ambas as faces, q₁ deve ser igual a q₂.");
          }
        }
        
        return errors.value.length === 0;
      };
  
      // Resolve o sistema linear T(x) = C1*x + C2
      const solveSystem = (eq1, eq2) => {
        const denominator = eq1.a * eq2.b - eq2.a * eq1.b;
        if (Math.abs(denominator) < 1e-15) {
          errors.value.push("Sistema singular: as condições podem não definir uma solução única.");
          return null;
        }
        return {
          C1: (eq2.b * eq1.c - eq1.b * eq2.c) / denominator,
          C2: (eq1.a * eq2.c - eq2.a * eq1.c) / denominator
        };
      };
  
      const calculate = () => {
        Q.value = Tx.value = null;
        if (!validateInputs()) return;
  
        try {
          let eq1, eq2;
          // Condição em x₁
          switch(innerBCType.value) {
            case 'temp':
              eq1 = boundaryConditions.temp(x1.value, innerTemp.value);
              break;
            case 'flux':
              eq1 = boundaryConditions.flux(x1.value, innerFlux.value, k.value);
              break;
            case 'conv':
              eq1 = boundaryConditions.conv(x1.value, innerH.value, innerTInf.value, k.value);
              break;
          }
          // Condição em x₂
          switch(outerBCType.value) {
            case 'temp':
              eq2 = boundaryConditions.temp(x2.value, outerTemp.value);
              break;
            case 'flux':
              eq2 = boundaryConditions.flux(x2.value, outerFlux.value, k.value);
              break;
            case 'conv':
              eq2 = boundaryConditions.conv(x2.value, outerH.value, outerTInf.value, k.value);
              break;
          }
          
          const solution = solveSystem(eq1, eq2);
          if (!solution) return;
          
          // Calcula Q: para placa plana, Q = -k * L * C1
          Q.value = -k.value * L.value * solution.C1;
          // Calcula a temperatura no ponto xInput: T(x) = C1*x + C2
          Tx.value = solution.C1 * xInput.value + solution.C2;
          
          // Aqui vai atualizar o gráfico
          updatePlot(solution.C1, solution.C2);
        } catch (error) {
          errors.value.push(`Erro: ${error.message}`);
        }
      };
  
      // Construcao do grafico
      const updatePlot = (C1, C2) => {
        const plotDiv = document.getElementById('plot');
        if (!plotDiv) return;
        const xValues = Array.from({ length: 100 }, (_, i) => {
          return x1.value + (i / 99) * (x2.value - x1.value);
        });
        const TValues = xValues.map(x => C1 * x + C2);
        Plotly.newPlot(plotDiv, [{
          x: xValues,
          y: TValues,
          type: 'scatter',
          mode: 'lines',
          line: { color: '#2196F3', width: 3 }
        }], {
          title: 'Distribuição de Temperatura na Placa',
          xaxis: { title: 'Posição (x) [m]' },
          yaxis: { title: 'Temperatura (°C)' }
        });
      };
  
      // Aqui ficam os observers e lifecycles
      watch([
        x1, x2, L, k,
        innerBCType, innerTemp, innerFlux, innerH, innerTInf,
        outerBCType, outerTemp, outerFlux, outerH, outerTInf,
        xInput
      ], calculate, { immediate: true });
  
      onMounted(calculate);
  
      return {
        x1, x2, L, k,
        innerBCType, innerTemp, innerFlux, innerH, innerTInf,
        outerBCType, outerTemp, outerFlux, outerH, outerTInf,
        xInput, Q, Tx, errors
      };
    }
};
</script>    

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  font-family: 'Segoe UI', sans-serif;
}

.input-group {
  margin-bottom: 15px;
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.input-group label {
  flex: 1 1 180px;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.input-section {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
  margin-bottom: 15px;
}

.results {
  margin-top: 20px;
  background: #e3f2fd;
  padding: 15px;
  border-radius: 8px;
}

.error {
  color: #d32f2f;
  background: #ffebee;
  padding: 12px;
  border-radius: 6px;
  margin: 15px 0;
  font-size: 0.9em;
}

#plot {
  width: 100%;
  height: 600px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
  margin-top: 20px;
}

.columns {
  display: grid;
  gap: 15px;
}

@media (min-width: 768px) {
  .columns {
    grid-template-columns: repeat(3, 1fr);
  }
}

@media (max-width: 767px) {
  .container {
    padding: 15px;
    width: 100%;
    box-sizing: border-box;
  }

  .columns {
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .input-section {
    width: 100%;
    max-width: 400px;
    margin: 0 auto 15px;
    padding: 20px;
  }

  .input-group label {
    flex: 1 1 100%;
  }

  input[type="number"], select {
    font-size: 16px;
    padding: 10px;
  }

  h2 {
    font-size: 24px;
    text-align: center;
  }

  h3 {
    font-size: 18px;
    text-align: center;
  }

  #plot {
    height: 300px;
    margin: 20px auto;
  }
}

input[type="number"] {
  padding: 8px 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  width: 100%;
  box-sizing: border-box;
}

select {
  padding: 8px 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  width: 100%;
  margin-bottom: 10px;
  font-size: 14px;
}

h2, h3 {
  color: #1a237e;
  margin-bottom: 15px;
}

h2 {
  font-size: 22px;
  border-bottom: 2px solid #1a237e;
  padding-bottom: 10px;
}

h3 {
  font-size: 16px;
}
</style>
