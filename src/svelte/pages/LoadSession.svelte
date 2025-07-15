<script lang="ts">
    import { onMount, onDestroy } from "svelte";
    import Chart from 'chart.js/auto';
    import { createPowerChart, updateChartData, updateMaxPowerLine, type ChartData } from '../lib/chartConfig';
    
    // Timer type alias for cross-platform compatibility
    type TimerHandle = ReturnType<typeof setTimeout>;
    
    export let title = "Tab";

    // Slider state
    let configError = false;
    let configurationLoaded = false;
    let kwValue: number = 11;
    let lastkWValue: number | null = null;
    let maxCurrentValue: number | null = null;
    let maxCurrentInterval: TimerHandle | null = null;
    let kwValueInterval: TimerHandle | null = null;
    let actualCDSPowerValue: number = 0;
    let actualCDSVoltageValue: number = 0;
    let actualCDSCurrentValue: number = 0; 
    let cdsPowerInterval: TimerHandle | null = null;
    let sinkPowerValue: any = null;
    let sinkPowerInterval: TimerHandle | null = null;

    // CSV Logging state
    let csvLoggingEnabled = false;
    let csvFilePath = '';

    // Auto Test state
    let autoTestRunning = false;
    let autoTestInterval: TimerHandle | null = null;
    let autoTestPhase: 'up' | 'down' = 'up'; // Track test direction
    
    // Soft start state
    let softStartRunning = false;
    let softStartInterval: TimerHandle | null = null;

    let currentHostIp: string = "";
    let voltageHostIP: string = "";
    let voltageLimit: number | null = null;
    let currentLimit: number | null = null;
    let maxkWValue: number | null = null;
    let maxkWValueExternal: number | null = null;
    let cds_ip: string = "";

    let setpointSent = false;
    let started = false;
    let isProcessing = false;

    // Chart data arrays
    let chart: Chart | null = null;
    let chartCanvas: HTMLCanvasElement;
    let timeLabels: string[] = [];
    let voltageHistory: number[] = [];
    let currentHistory: number[] = [];
    let powerHistory: number[] = []; // Power in kW


    onMount(async () => {
        
        
        // Initialize chart first (before config loading)
        const initialChartData: ChartData = {
            timeLabels,
            voltageData: voltageHistory,
            currentData: currentHistory,
            powerData: powerHistory,
            maxPowerLimit: null // Will be updated after config loads
        };
        chart = createPowerChart(chartCanvas, initialChartData);
        
        // Initialize CSV logging status
        await updateCsvLoggingStatus();
        
        try {
            const config = await globalThis.api.invoke('getConfigFromFile');
            currentHostIp = config.currenthostIP || "";
            voltageHostIP = config.voltagehostIP || "";        
            voltageLimit = config.voltageLimit ?? null;
            currentLimit = config.currentLimit ?? null;
            maxkWValue = config.maxkWValue ?? null;
            cds_ip = config.cdsIP || "";

            await globalThis.api.invoke("setVoltagePriorityMode", { HostIp: voltageHostIP, OutputVoltageLimitCV: voltageLimit, OutputCurrentLimitCV: currentLimit });
            await globalThis.api.invoke("setCurrentPriorityMode", { HostIp: currentHostIp, OutputCurrentLimitCC: currentLimit !== null ? currentLimit : null, OutputVoltageLimitCC: voltageLimit !== null ? voltageLimit + 10 : null });
            
            configurationLoaded = true;
            configError = false;
            
            // Update chart with loaded config values
            if (chart) {
                updateMaxPowerLine(chart, maxkWValue, timeLabels.length);
            }
            
            await globalThis.api.invoke("setCurrentSetPoint", { HostIp: currentHostIp, CurrentSetPoint: 1 });
            maxCurrentInterval = setInterval(pollMaxCurrentValue, 2000);
            pollMaxCurrentValue(); // Initialer Aufruf

            // Start Backend CDS Polling first
            await globalThis.api.invoke("startPollingCDS", {});
            
            // Single unified CDS polling interval (gets data from backend cache)
            cdsPowerInterval = setInterval(async () => {
                try {
                    // Get all CDS values in one batch
                    const [powerResult, voltageResult, currentResult] = await Promise.all([
                        globalThis.api.invoke("getLastPowerValueCDS"),
                        globalThis.api.invoke("getLastVoltageValueCDS"), 
                        globalThis.api.invoke("getLastCurrentValueCDS")
                    ]);

                    // Debug logging
                    console.log("CDS Raw Results:", {
                        power: powerResult,
                        voltage: voltageResult,
                        current: currentResult,
                        powerType: typeof powerResult,
                        voltageType: typeof voltageResult,
                        currentType: typeof currentResult
                    });

                    // Update power value
                    if (powerResult !== null && powerResult !== undefined && !isNaN(Number(powerResult))) {
                        const newPower = Number(powerResult);
                        if (newPower !== actualCDSPowerValue) {
                            console.log(`Power updated: ${actualCDSPowerValue} → ${newPower}`);
                            actualCDSPowerValue = newPower;
                        }
                    } else {
                        console.log("Power result invalid:", powerResult);
                    }

                    // Update voltage value  
                    if (voltageResult !== null && voltageResult !== undefined && !isNaN(Number(voltageResult))) {
                        const newVoltage = Number(voltageResult);
                        if (newVoltage > 100 && newVoltage !== actualCDSVoltageValue) {
                            console.log(`Voltage updated: ${actualCDSVoltageValue} → ${newVoltage}`);
                            actualCDSVoltageValue = newVoltage;
                        }
                    } else {
                        console.log("Voltage result invalid:", voltageResult);
                    }

                    // Update current value
                    if (currentResult !== null && currentResult !== undefined && !isNaN(Number(currentResult))) {
                        const newCurrent = Number(currentResult);
                        if (newCurrent > 0 && newCurrent !== actualCDSCurrentValue) {
                            console.log(`Current updated: ${actualCDSCurrentValue} → ${newCurrent}`);
                            actualCDSCurrentValue = newCurrent;
                        }
                    } else {
                        console.log("Current result invalid:", currentResult);
                    }

                    // Update chart with all new data
                    const powerKw = actualCDSPowerValue / 1000;
                    const voltage = (actualCDSVoltageValue > 100) ? actualCDSVoltageValue : 0;
                    const current = actualCDSCurrentValue;

                    console.log("Chart Data:", { powerKw, voltage, current });

                    const now = new Date().toLocaleTimeString();
                    
                    // Für Auto Test: Alle Datenpunkte speichern, sonst nur die letzten 50
                    if (!autoTestRunning && timeLabels.length >= 50) {
                        timeLabels.shift();
                        voltageHistory.shift();
                        currentHistory.shift();
                        powerHistory.shift();
                    }
                    
                    timeLabels.push(now);
                    voltageHistory.push(voltage);
                    currentHistory.push(current);
                    powerHistory.push(powerKw);
                    
                    // Update chart with new data
                    if (chart) {
                        const chartData: ChartData = {
                            timeLabels,
                            voltageData: voltageHistory,
                            currentData: currentHistory,
                            powerData: powerHistory,
                            maxPowerLimit: maxkWValue
                        };
                        updateChartData(chart, chartData);
                        console.log("Chart updated");
                    } else {
                        console.log("Chart is null");
                    }

                } catch (error) {
                    console.error("CDS polling error:", error);
                    // Don't reset values on error, keep last valid data
                }
            }, 2000); // Erhöht auf 2 Sekunden

            //Sink Power Polling
            await globalThis.api.invoke("startSinkPowerPolling", { ip: currentHostIp, intervalMs: 1000, timeoutMs: 3000 });
            sinkPowerInterval = setInterval(async () => {
            try {
              const result = await globalThis.api.invoke("getLastSinkPowerValue");
              sinkPowerValue = result;
            } catch {
                sinkPowerValue = null;
         }
            }, 2000);
            
        } catch (err) {
            configurationLoaded = false;
            configError = true;
            console.error("Config error:", err);
        }
    })

    onDestroy(() => {
        // Cleanup page visibility listener
        document.removeEventListener('visibilitychange', () => {});
        
        // Stop all intervals first
        if (maxCurrentInterval) {
            clearInterval(maxCurrentInterval);
            maxCurrentInterval = null;
        }
        if (cdsPowerInterval) {
            clearInterval(cdsPowerInterval);
            cdsPowerInterval = null;
        }
        if (sinkPowerInterval) {
            clearInterval(sinkPowerInterval);
            sinkPowerInterval = null;
        }
        if (kwValueInterval) {
            clearInterval(kwValueInterval);
            kwValueInterval = null;
        }

        // Stop backend polling immediately (fire-and-forget)
        globalThis.api.invoke("stopPollingCDS").catch(error => {
            console.error("Error stopping CDS polling:", error);
        });

        globalThis.api.invoke("stopSinkPowerPolling").catch(error => {
            console.error("Error stopping Sink Power polling:", error);
        });

        // Stop auto test if running
        if (autoTestInterval) {
            clearInterval(autoTestInterval);
            autoTestInterval = null;
        }
        
        // Stop soft start if running
        if (softStartInterval) {
            clearInterval(softStartInterval);
            softStartInterval = null;
        }

        // Destroy chart last
        if (chart) {
            chart.destroy();
            chart = null;
        }
    })

    
    async function pollMaxCurrentValue() {
    try {
        const result = await globalThis.api.invoke("readEvDuty", { ip: cds_ip });
        console.log("MaxCurrent result:", result);
        if (result && !isNaN(Number(result))) {
            maxCurrentValue = Number(Number(result*0.6).toFixed(2));
            maxkWValueExternal = Number(((maxCurrentValue * voltageLimit) / 1000).toFixed(2));
            // Update max power line in chart
            if (chart) {
                updateMaxPowerLine(chart, maxkWValue, timeLabels.length);
            }
        } else {
            maxCurrentValue = null;
        }
    } catch (err) {
        console.error("MaxCurrent error:", err);
        maxCurrentValue = null;
    }
}

    

    // Der neue Schalter-Button mit Soft Start
    async function handleToggleOutput() {
        if (isProcessing || !configurationLoaded) return;
        isProcessing = true;
        try {
            if (!started) {
                // Output aktivieren
                await globalThis.api.invoke("setOutput", { HostIp: currentHostIp, OutputState: true });
                await globalThis.api.invoke("setOutput", { HostIp: voltageHostIP, OutputState: true });
                started = true;
                
                // Soft Start: Langsam von 1 kW auf gewünschten Wert hochfahren
                await startSoftStart();
            } else {
                // Soft Start stoppen falls läuft
                if (softStartInterval) {
                    clearInterval(softStartInterval);
                    softStartInterval = null;
                    softStartRunning = false;
                }
                
                // Output deaktivieren
                await globalThis.api.invoke("setOutput", { HostIp: currentHostIp, OutputState: false });
                await globalThis.api.invoke("setOutput", { HostIp: voltageHostIP, OutputState: false });
                started = false;
                setpointSent = false;
            }
        } finally {
            isProcessing = false;
        }
    }
 $: if (started && !softStartRunning && !autoTestRunning) {
       if (!kwValueInterval) {
        kwValueInterval = setInterval(() => {
                        handleSendSetpoint();            
        }, 100); 
      }
   } else {
       if (kwValueInterval) {
           clearInterval(kwValueInterval);
           kwValueInterval = null;
       }
    }    
    
    async function handleSendSetpoint() {
    if (isProcessing || !configurationLoaded) return;
    isProcessing = true;
    try {
        let setpointToSend = kwValue;
        if (maxkWValueExternal !== null && kwValue > maxkWValueExternal) {
            setpointToSend = maxkWValueExternal;
        }
        if (lastkWValue !== setpointToSend) {
            const currentSetPoint = await globalThis.api.invoke("defineVoltageCurrent", { powerInkW: setpointToSend, voltage: voltageLimit });
            await globalThis.api.invoke("setCurrentSetPoint", { HostIp: currentHostIp, CurrentSetPoint: currentSetPoint.current });
            lastkWValue = setpointToSend;
            setpointSent = true;
        }
    } finally {
        isProcessing = false;
    }
}

// Soft Start Functions
async function startSoftStart() {
    if (softStartRunning) return;
    
    softStartRunning = true;
    const targetPower = kwValue;
    let currentSoftPower = 1.0; // Start bei 1 kW
    const step = 0.5; // 0.5 kW Schritte
    const stepTime = 200; // 200ms pro Schritt = sanfter Anstieg
    
    console.log(`Soft Start: Ramping from 1.0 kW to ${targetPower} kW`);
    
    const rampStep = async () => {
        if (softStartRunning && started) {
            // Setpoint senden
            try {
                let setpointToSend = currentSoftPower;
                if (maxkWValueExternal !== null && currentSoftPower > maxkWValueExternal) {
                    setpointToSend = maxkWValueExternal;
                }
                
                const currentSetPoint = await globalThis.api.invoke("defineVoltageCurrent", { 
                    powerInkW: setpointToSend, 
                    voltage: voltageLimit 
                });
                await globalThis.api.invoke("setCurrentSetPoint", { 
                    HostIp: currentHostIp, 
                    CurrentSetPoint: currentSetPoint.current 
                });
                
                lastkWValue = setpointToSend;
                setpointSent = true;
                
                console.log(`Soft Start: Ramped to ${setpointToSend} kW`);
            } catch (error) {
                console.error('Soft Start setpoint error:', error);
            }
            
            // Nächster Schritt
            if (currentSoftPower < targetPower) {
                currentSoftPower = Math.min(currentSoftPower + step, targetPower);
                softStartInterval = setTimeout(rampStep, stepTime);
            } else {
                // Soft Start beendet - normales Setpoint-Interval übernimmt
                console.log('Soft Start completed - switching to normal operation');
                softStartRunning = false;
                softStartInterval = null;
            }
        }
    };
    
    // Ersten Schritt starten
    rampStep();
}

// CSV Logging Functions
async function toggleCsvLogging() {
    try {
        if (csvLoggingEnabled) {
            const result = await globalThis.api.invoke('stopCsvLogging');
            if (result.success) {
                csvLoggingEnabled = false;
                csvFilePath = '';
                console.log('CSV logging stopped. File saved:', result.filePath);
            }
        } else {
            const result = await globalThis.api.invoke('startCsvLogging');
            if (result.success) {
                csvLoggingEnabled = true;
                csvFilePath = result.filePath;
                console.log('CSV logging started. File:', result.filePath);
            }
        }
    } catch (error) {
        console.error('Error toggling CSV logging:', error);
    }
}

async function updateCsvLoggingStatus() {
    try {
        const status = await globalThis.api.invoke('getCsvLoggingStatus');
        csvLoggingEnabled = status.enabled;
        csvFilePath = status.filePath;
    } catch (error) {
        console.error('Error getting CSV status:', error);
    }
}

// Auto Test Functions
function clearChartForAutoTest() {
    // Chart-Daten leeren für saubere Auto Test Aufzeichnung
    timeLabels.length = 0;
    voltageHistory.length = 0;
    currentHistory.length = 0;
    powerHistory.length = 0;
    
    // Chart sofort aktualisieren
    if (chart) {
        const chartData: ChartData = {
            timeLabels,
            voltageData: voltageHistory,
            currentData: currentHistory,
            powerData: powerHistory,
            maxPowerLimit: maxkWValue
        };
        updateChartData(chart, chartData);
        console.log('Chart cleared for Auto Test');
    }
}

async function startAutoTest() {
    if (autoTestRunning || !configurationLoaded) return;
    
    autoTestRunning = true;
    
    try {
        // Chart für saubere Aufzeichnung leeren
        clearChartForAutoTest();
        
        // 1. Ladevorgang aktivieren
        await globalThis.api.invoke("setOutput", { HostIp: currentHostIp, OutputState: true });
        await globalThis.api.invoke("setOutput", { HostIp: voltageHostIP, OutputState: true });
        started = true;
        
        // 2. Sanfter Start auf 1 kW (verhindert Stromspitze)
        try {
            const currentSetPoint = await globalThis.api.invoke("defineVoltageCurrent", { 
                powerInkW: 1.0, 
                voltage: voltageLimit 
            });
            await globalThis.api.invoke("setCurrentSetPoint", { 
                HostIp: currentHostIp, 
                CurrentSetPoint: currentSetPoint.current 
            });
            console.log('Auto Test: Set initial 1 kW to prevent current spike');
        } catch (error) {
            console.error('Auto Test initial setpoint error:', error);
        }
        
        // 3. Warten auf 3 Sekunden für Stabilisierung
        await new Promise(resolve => setTimeout(resolve, 3000));
        
        // 4. Von 1 kW bis 21 kW in 0.1 kW Schritten, dann zurück - JEDER WERT 3 SEKUNDEN HALTEN
        let currentPower = 1.0;
        const maxPower = 21.0;
        const minPower = 1.0;
        const step = 0.1;
        autoTestPhase = 'up'; // Start mit Hochfahren
        
        const testStep = async () => {
            if (autoTestRunning) {
                kwValue = Number(currentPower.toFixed(1));
                
                // Setpoint senden
                try {
                    let setpointToSend = kwValue;
                    if (maxkWValueExternal !== null && kwValue > maxkWValueExternal) {
                        setpointToSend = maxkWValueExternal;
                    }
                    
                    const currentSetPoint = await globalThis.api.invoke("defineVoltageCurrent", { 
                        powerInkW: setpointToSend, 
                        voltage: voltageLimit 
                    });
                    await globalThis.api.invoke("setCurrentSetPoint", { 
                        HostIp: currentHostIp, 
                        CurrentSetPoint: currentSetPoint.current 
                    });
                    
                    console.log(`Auto Test (${autoTestPhase}): Set power to ${setpointToSend} kW - holding for 3 seconds`);
                } catch (error) {
                    console.error('Auto Test setpoint error:', error);
                }
                
                // Bestimme nächsten Wert basierend auf Phase
                if (autoTestPhase === 'up') {
                    if (currentPower < maxPower) {
                        currentPower += step;
                    } else {
                        // Am Maximum angekommen - wechsle zu Rückfahrt
                        autoTestPhase = 'down';
                        currentPower -= step;
                        console.log('Auto Test: Switching to DOWN phase at maximum power');
                    }
                } else { // autoTestPhase === 'down'
                    if (currentPower > minPower) {
                        currentPower -= step;
                    } else {
                        // Am Minimum angekommen - Test beendet
                        console.log('Auto Test: Reached minimum power - test complete');
                        await stopAutoTest();
                        return; // Wichtig: testStep nicht weiter aufrufen
                    }
                }
                
                // 3 Sekunden warten bevor der nächste Schritt
                autoTestInterval = setTimeout(testStep, 3000);
            }
        };
        
        // Ersten Schritt starten
        testStep();
        
    } catch (error) {
        console.error('Auto Test error:', error);
        autoTestRunning = false;
    }
}

async function stopAutoTest() {
    if (autoTestInterval) {
        clearTimeout(autoTestInterval);
        autoTestInterval = null;
    }
    
    try {
        // Ladevorgang deaktivieren
        await globalThis.api.invoke("setOutput", { HostIp: currentHostIp, OutputState: false });
        await globalThis.api.invoke("setOutput", { HostIp: voltageHostIP, OutputState: false });
        started = false;
        setpointSent = false;
        
        // kW Wert auf 1 zurücksetzen
        kwValue = 1.0;
        
        console.log('Auto Test completed and output deactivated');
    } catch (error) {
        console.error('Error stopping auto test:', error);
    } finally {
        autoTestRunning = false;
        autoTestPhase = 'up'; // Reset für nächsten Test
    }
}


 
   
</script>

<div>
    <!--<h1>{title}!</h1>-->
    <div class="slider-container">
    <label for="kw-slider">Power Setpoint (kW)</label>
    <input
        id="kw-slider"
        type="range"
        min="1"
        max={maxkWValue !== null ? maxkWValue : 22}
        step="0.1"
        bind:value={kwValue}
    />
    <span class="kw-value">{kwValue.toFixed(1)} kW</span>
    <div class="button-row">
    <button
        class="toggle-btn {started ? 'deactivate' : ''}"
        title={started ? "Deactivate Output" : "Activate Output"}
        on:click={handleToggleOutput}
        disabled={!configurationLoaded}    >
        {#if started}
            <i class="fa fa-power-off"></i> Deactivate
        {:else}
            <i class="fa fa-bolt"></i> Activate
        {/if}
    </button>
    
    <!-- CSV Logging Controls -->
    <button
        class="toggle-btn csv-btn {csvLoggingEnabled ? 'csv-active' : ''}"
        title={csvLoggingEnabled ? "Stop CSV Logging" : "Start CSV Logging"}
        on:click={toggleCsvLogging}
        disabled={!configurationLoaded}>
        {#if csvLoggingEnabled}
            <i class="fa fa-stop"></i> Stop CSV Logging
        {:else}
            <i class="fa fa-file-csv"></i> Start CSV Logging
        {/if}
    </button>
    
    <!-- Auto Test Button -->
    <button
        class="toggle-btn auto-test-btn {autoTestRunning ? 'test-active' : ''}"
        title={autoTestRunning ? "Auto Test Running..." : "Start Auto Test (1-21-1 kW)"}
        on:click={autoTestRunning ? stopAutoTest : startAutoTest}
        disabled={!configurationLoaded || started}>
        {#if autoTestRunning}
            <i class="fa fa-spinner fa-spin"></i> Stop Test
        {:else}
            <i class="fa fa-play"></i> Auto Test
        {/if}
    </button>
</div>

{#if csvLoggingEnabled && csvFilePath}
    <div class="csv-status">
        <i class="fa fa-file-csv"></i>
        <span>Recording to: {csvFilePath.split('\\').pop()}</span>
    </div>
{/if}

{#if autoTestRunning}
    <div class="auto-test-status">
        <i class="fa fa-cog fa-spin"></i>
        <span>Auto Test Running ({autoTestPhase === 'up' ? '↗' : '↙'}): {kwValue.toFixed(1)} kW</span>
    </div>
{/if}
    {#if !configurationLoaded && !configError}
        <div class="hint">
            
            <strong>Please Wait for Configuration.</strong>
            
        </div>
        <div>
            <i class="fa fa-hourglass-half fa-spin" style="margin-right:0.7em; font-size:2.2em; vertical-align:middle;"></i>
        </div>
    {:else if configError }
        <div class="hint">
            <strong>Error loading configuration. Please check your Settings and Setup.</strong>
        </div>
    {/if}
</div>
    <div class="actual-value">
    <table class="value-table centered-table">
        <tr>
            <th>CDS Power Value</th>
            <th>CDS Voltage L1</th>
            <th>Sink Power Value</th>
            <th>Max Current (EVSE)</th>
            <th>Max Power (EVSE)</th>
        </tr>
        <tr>
            <td>
                {#if actualCDSPowerValue > 0}
                    {(actualCDSPowerValue / 1000).toFixed(2)} kW
                {:else}
                0.00 kW
            {/if}
            </td>
            <td>
                {#if actualCDSVoltageValue > 100}
                    {actualCDSVoltageValue.toFixed(1)} V
                {:else}
                0.0 V
            {/if}
            </td>
            <td>
            {#if sinkPowerValue && sinkPowerValue.successful}
                {(Number(sinkPowerValue.msg) / 1000).toFixed(2)} kW
            {:else}
                n/a
            {/if}
        </td>
            <td>
                {maxCurrentValue !== null ? maxCurrentValue.toFixed(2) + ' A' : 'n/a'}
            </td>
            <td>
                {maxkWValueExternal !== null ? maxkWValueExternal.toFixed(2) + ' kW' : 'n/a'}
            </td>
        </tr>
    </table>
</div>

<div class="power-graph-container" style="margin-top:1.5em;">
    <canvas bind:this={chartCanvas} width="1200" height="450"></canvas>
</div>

</div>


<style>
    @import url('https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css');

    div {
        padding-top: 10px;
        padding-bottom: 10px;
        text-align: center;
        background-color: rgba(255, 255, 255, 0.5);
    }
   
    .slider-container {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        margin: 30px auto 20px auto;
        width: 350px;
    }

    .slider-container label {
        font-size: 1.2em;
        margin-bottom: 8px;
    }

    #kw-slider {
        width: 100%;
        margin: 10px 0;
    }

    .kw-value {
        font-size: 1.5em;
        font-weight: bold;
        margin-top: 5px;
        color: #333;
    }

    .button-row {
        display: flex;
        gap: 12px;
        margin-top: 18px;
        justify-content: center;
    }   
 
     .hint {
        color: #b00;
        font-weight: bold;
        text-align: center;
        margin-top: 1.2em;
        font-size: 1.15em;
        background: #fff3f3;
        border: 2px solid #b00;
        border-radius: 6px;
        padding: 0.7em 1em;
        box-shadow: 0 0 8px #b00a;
        letter-spacing: 0.5px;
    }   

    .actual-value {
        margin-top: 20px;
        font-size: 1.2em;
        font-weight: 500;
    }
   .toggle-btn {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 24px;
    font-size: 1.2em;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    color: #fff;
    font-weight: bold;
    transition: background 0.2s, color 0.2s;
    margin: 0 auto;
    min-width: 180px;
    justify-content: center;
    background-color: #43a047; /* Standard: grün für Activate */
}
.toggle-btn:disabled {
    background-color: #bdbdbd;
    color: #fff;
    cursor: not-allowed;
}
.toggle-btn:hover:enabled {
    filter: brightness(0.95);
}
.toggle-btn.deactivate {
    background-color: #ff0000; /* Rot für Deactivate */
}

.toggle-btn.csv-btn {
    background-color: #28a745; /* Grün für CSV */
    margin-left: 0.5em;
}

.toggle-btn.csv-active {
    background-color: #dc3545; /* Rot wenn CSV aktiv (Stop) */
}

.toggle-btn.auto-test-btn {
    background-color: #007bff; /* Blau für Auto Test */
    margin-left: 0.5em;
}

.toggle-btn.test-active {
    background-color: #ffc107; /* Gelb wenn Test läuft */
    color: #212529;
}

.csv-status {
    background-color: #d4edda;
    border: 1px solid #c3e6cb;
    border-radius: 6px;
    color: #155724;
    padding: 0.5em;
    margin-top: 0.5em;
    font-size: 0.9em;
    text-align: center;
}

.csv-status i {
    margin-right: 0.5em;
    color: #28a745;
}

.auto-test-status {
    background-color: #fff3cd;
    border: 1px solid #ffeaa7;
    border-radius: 6px;
    color: #856404;
    padding: 0.5em;
    margin-top: 0.5em;
    font-size: 0.9em;
    text-align: center;
}

.auto-test-status i {
    margin-right: 0.5em;
    color: #007bff;
}

.power-graph-container {
    margin: 2em auto 0 auto;
    max-width: 1200px;
    width: 95%;
    background: #fff;
    border-radius: 10px;
    box-shadow: 0 2px 12px #0077cc22;
    padding: 1.5em 2em;
    min-height: 500px;
}

.power-graph-container canvas {
    width: 100% !important;
    height: 450px !important;
    max-width: 100%;
}

.value-table {
    border-collapse: collapse;
    margin-bottom: 0.5em;
    width: 100%;
    max-width: 600px;
}
.value-table th, .value-table td {
    border: 1px solid #ddd;
    padding: 0.5em 1em;
    text-align: center;
}
.value-table th {
    background: #f5f5f5;
    font-weight: 600;
}
.centered-table {
    margin-left: auto;
    margin-right: auto;
}

</style>
