<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <title>Suivi des Macros - Avancé</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    body { font-family: Arial; padding: 20px; max-width: 700px; margin: auto; }
    .bar { height: 20px; background: #eee; border-radius: 5px; margin-bottom: 10px; overflow: hidden; }
    .bar-fill { height: 100%; color: white; text-align: right; padding-right: 5px; font-size: 12px; font-weight: bold; }
    .calories { background: #ff6666; }
    .proteins { background: #66cc66; }
    .fats { background: #ffcc66; }
    .carbs { background: #66b3ff; }
    .fibers { background: #cc99ff; }
    input, select { margin: 5px 0; padding: 5px; width: 100%; }
    button { margin: 10px 5px 10px 0; padding: 8px 12px; }
    .meal-item { border: 1px solid #ddd; padding: 10px; border-radius: 5px; margin-bottom: 10px; }
  </style>
</head>
<body>
  <h1>Suivi des Macros (par jour)</h1>
  <label for="date">Date : </label>
  <input type="date" id="date" value="">

  <div id="macros-bars"></div>
  <div id="alert"></div>

  <h2>Ajouter un repas</h2>
  <form id="meal-form">
    <input type="text" id="meal-name" placeholder="Nom du repas" required>
    <input type="number" id="calories" placeholder="Calories" step="1">
    <input type="number" id="proteins" placeholder="Protéines (g)" step="0.1">
    <input type="number" id="fibers" placeholder="Fibres (g)" step="0.1">
    <input type="number" id="fats" placeholder="Lipides (g)" step="0.1">
    <input type="number" id="carbs" placeholder="Glucides (g)" step="0.1">
    <button type="submit">Ajouter le repas</button>
  </form>

  <h2>Historique des repas</h2>
  <div id="meal-list"></div>

  <h2>Suivi du poids</h2>
  <form id="weight-form">
    <input type="number" id="weight-input" placeholder="Poids (kg)" step="0.1">
    <button type="submit">Ajouter le poids</button>
  </form>
  <canvas id="weightChart" width="400" height="200"></canvas>

  <script>
    const goals = { calories: 2000, proteins: 136, fibers: 36, fats: 130, carbs: 130 };
    const alertThresholds = { proteins: 30, calories: 500 };

    let data = JSON.parse(localStorage.getItem("macroData")) || {};
    let weightHistory = JSON.parse(localStorage.getItem("weightHistory")) || [];

    document.getElementById("date").value = new Date().toISOString().split('T')[0];

    function getToday() {
      return document.getElementById("date").value;
    }

    function getMeals() {
      const today = getToday();
      return data[today]?.meals || [];
    }

    function saveData() {
      localStorage.setItem("macroData", JSON.stringify(data));
    }

    function saveWeight() {
      localStorage.setItem("weightHistory", JSON.stringify(weightHistory));
    }

    function updateBars() {
      const today = getToday();
      const meals = getMeals();
      const totals = { calories: 0, proteins: 0, fibers: 0, fats: 0, carbs: 0 };
      meals.forEach(m => {
        for (let key in totals) totals[key] += m[key];
      });
      const remaining = {};
      for (let key in goals) {
        remaining[key] = goals[key] - totals[key];
      }
      const bars = Object.keys(goals).map(key => {
        const left = remaining[key] < 0 ? 0 : remaining[key];
        const percent = Math.max(0, Math.min(100, (left / goals[key]) * 100));
        return `
          <div>
            <label>${key.charAt(0).toUpperCase() + key.slice(1)}: ${left.toFixed(1)}${key === 'calories' ? '' : 'g'} restants</label>
            <div class="bar">
              <div class="bar-fill ${key}" style="width:${percent}%">${left.toFixed(0)}${key === 'calories' ? '' : 'g'}</div>
            </div>
          </div>`;
      }).join("");
      document.getElementById("macros-bars").innerHTML = bars;

      // Alerts
      let alert = "";
      for (let key in alertThresholds) {
        if (remaining[key] < alertThresholds[key]) {
          alert += `<p style='color: red;'>⚠️ Il te reste peu de ${key} : ${remaining[key].toFixed(0)}${key === 'calories' ? '' : 'g'}</p>`;
        }
      }
      document.getElementById("alert").innerHTML = alert;
    }

    function renderMeals() {
      const today = getToday();
      const container = document.getElementById("meal-list");
      const meals = getMeals();
      container.innerHTML = meals.map((meal, i) => `
        <div class="meal-item">
          <strong>${meal.name}</strong><br>
          ${meal.calories} kcal, ${meal.proteins}g prot, ${meal.fibers}g fibres, ${meal.fats}g lip, ${meal.carbs}g gluc<br>
          <button onclick="deleteMeal(${i})">🗑️ Supprimer</button>
        </div>
      `).join("");
    }

    function deleteMeal(index) {
      const today = getToday();
      data[today].meals.splice(index, 1);
      saveData();
      updateBars();
      renderMeals();
    }

    document.getElementById("meal-form").addEventListener("submit", function(e) {
      e.preventDefault();
      const today = getToday();
      data[today] = data[today] || { meals: [] };
      const meal = {
        name: document.getElementById("meal-name").value,
        calories: parseFloat(document.getElementById("calories").value) || 0,
        proteins: parseFloat(document.getElementById("proteins").value) || 0,
        fibers: parseFloat(document.getElementById("fibers").value) || 0,
        fats: parseFloat(document.getElementById("fats").value) || 0,
        carbs: parseFloat(document.getElementById("carbs").value) || 0
      };
      data[today].meals.push(meal);
      saveData();
      this.reset();
      updateBars();
      renderMeals();
    });

    document.getElementById("weight-form").addEventListener("submit", function(e) {
      e.preventDefault();
      const date = getToday();
      const weight = parseFloat(document.getElementById("weight-input").value);
      if (!isNaN(weight)) {
        weightHistory.push({ date, weight });
        saveWeight();
        renderChart();
        this.reset();
      }
    });

    function renderChart() {
      const ctx = document.getElementById("weightChart").getContext("2d");
      const sorted = weightHistory.sort((a, b) => a.date.localeCompare(b.date));
      const chart = new Chart(ctx, {
        type: 'line',
        data: {
          labels: sorted.map(e => e.date),
          datasets: [{
            label: 'Poids (kg)',
            data: sorted.map(e => e.weight),
            borderColor: '#007bff',
            backgroundColor: 'rgba(0,123,255,0.1)',
            fill: true
          }]
        },
        options: {
          responsive: true,
          plugins: { legend: { display: true } }
        }
      });
    }

    updateBars();
    renderMeals();
    renderChart();
  </script>
</body>
</html>

