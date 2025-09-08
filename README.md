<!DOCTYPE html>
<html>
<head>
    <title>Function Analysis</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.10.1/math.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>
    <h1>Function Analysis</h1>
    <label for="functionInput">Enter function (e.g., x^2 + 2*x - 1):</label>
    <input type="text" id="functionInput" name="functionInput" value="x^2">
    <button onclick="analyzeFunction()">Analyze</button>

    <h2>Domain:</h2>
    <p id="domainOutput"></p>

    <h2>Range:</h2>
    <p id="rangeOutput"></p>

    <h2>Graph:</h2>
    <canvas id="graphCanvas" width="400" height="300"></canvas>

    <script>
        function analyzeFunction() {
            var functionInput = document.getElementById("functionInput").value;

            try {
                // Compile the expression using Math.js
                const node = math.parse(functionInput);
                const code = node.compile();

                let xValues = [];
                let yValues = [];
                let minY = Infinity; // For range calculation
                let maxY = -Infinity;

                for (let x = -10; x <= 10; x += 0.5) {
                    xValues.push(x);
                    try {
                        const y = code.evaluate({ x: x });
                        yValues.push(y);

                        if (typeof y === 'number' && !isNaN(y)) {
                            minY = Math.min(minY, y);
                            maxY = Math.max(maxY, y);
                        }

                    } catch (error) {
                        console.error("Error evaluating function at x =", x, error);
                        yValues.push(NaN);
                    }
                }

                let domain = "(-∞, ∞)"; // Simplified (improve for specific functions)
                let range = "[" + (isFinite(minY) ? minY.toFixed(2) : "Unknown") + ", " + (isFinite(maxY) ? maxY.toFixed(2) : "Unknown") + "]";

                document.getElementById("domainOutput").innerText = "Domain: " + domain;
                document.getElementById("rangeOutput").innerText = "Range: " + range;

                // Destroy the previous chart, if it exists
                if (window.myChart) {
                    window.myChart.destroy();
                }

                // Create the chart
                const ctx = document.getElementById('graphCanvas').getContext('2d');
                window.myChart = new Chart(ctx, {
                    type: 'line',
                    data: {
                        labels: xValues,
                        datasets: [{
                            label: functionInput || 'f(x)',
                            data: yValues,
                            borderColor: 'rgb(75, 192, 192)',
                            tension: 0.1
                        }]
                    },
                    options: {
                        scales: {
                            x: {
                                title: {
                                    display: true,
                                    text: 'x'
                                }
                            },
                            y: {
                                title: {
                                    display: true,
                                    text: 'f(x)'
                                }
                            }
                        }
                    }
                });


            } catch (error) {
                alert("Invalid function: " + error);
                console.error("Function parsing error:", error);
                document.getElementById("domainOutput").innerText = "";
                document.getElementById("rangeOutput").innerText = "";
            }
        }
    </script>
</body>
</html>
