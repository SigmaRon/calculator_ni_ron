<!DOCTYPE html>
<html>
<head>
    <title>Domain, Range, and Graph</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>
    <h1>Function Analysis</h1>
    <label for="functionInput">Enter function:</label>
    <input type="text" id="functionInput" name="functionInput">
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

            // Basic example function (you'll need a proper parser)
            let xValues = [];
            let yValues = [];
            for (let x = -10; x <= 10; x += 0.5) {
                xValues.push(x);
                // Example: f(x) = x^2
                yValues.push(x * x);
            }

            // Create the chart
            new Chart("graphCanvas", {
                type: "line",
                data: {
                    labels: xValues,
                    datasets: [{
                        label: functionInput || 'f(x) = x^2',
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
            
            document.getElementById("domainOutput").innerText = "Domain: (-∞, ∞)";
            document.getElementById("rangeOutput").innerText = "Range: [0, ∞)"; // for f(x) = x^2
        }
    </script>
</body>
</html>
