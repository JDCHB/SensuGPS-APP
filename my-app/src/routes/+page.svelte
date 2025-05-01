<script>
    import { onMount, onDestroy } from "svelte";
    import { Geolocation } from "@capacitor/geolocation";
    import { Capacitor } from "@capacitor/core";

    let lat = "";
    let lon = "";
    let error = null;
    let intervalo = null;
    const discapacitado_id = 12;
    let isMobile = false;

    onMount(async () => {
        // Verificar si es móvil
        isMobile = Capacitor.isNativePlatform();

        async function obtenerUbicacionYEnviar() {
            try {
                // Configuración específica para móvil
                const options = isMobile
                    ? {
                          enableHighAccuracy: true,
                          timeout: 10000,
                          maximumAge: 0,
                      }
                    : {};

                const position = await Geolocation.getCurrentPosition(options);

                lat = position.coords.latitude.toString();
                lon = position.coords.longitude.toString();

                // Validación reforzada para móviles
                if (!validarCoordenadas(lat, lon)) {
                    throw {
                        message: "Coordenadas inválidas en móvil",
                        details: { lat, lon, isMobile },
                    };
                }

                const res = await fetch(
                    `https://proyectomascotas.onrender.com/update_coordenadas_discapacitado/${discapacitado_id}`,
                    {
                        method: "PUT",
                        headers: {
                            "Content-Type": "application/json",
                            Accept: "application/json",
                        },
                        body: JSON.stringify({
                            latitud: lat,
                            longitud: lon,
                        }),
                    },
                );

                if (!res.ok) throw await procesarError(res);

                console.log("Ubicación actualizada correctamente");
                error = null;
            } catch (err) {
                error = {
                    message: err.message || "Error en móvil",
                    details: err.details || err,
                };
                console.error(
                    "Error en móvil:",
                    JSON.stringify(error, null, 2),
                );

                // Intento alternativo si falla GPS
                if (isMobile && err.message?.includes("Timeout")) {
                    await obtenerUbicacionAlternativa();
                }
            }
        }

        async function obtenerUbicacionAlternativa() {
            try {
                console.warn("Intentando obtener ubicación por red...");
                const position = await Geolocation.getCurrentPosition({
                    enableHighAccuracy: false, // Usa red celular/WiFi
                });
                lat = position.coords.latitude.toString();
                lon = position.coords.longitude.toString();
            } catch (altErr) {
                console.error("Error alternativo:", altErr);
            }
        }

        function validarCoordenadas(lat, lon) {
            const latNum = parseFloat(lat);
            const lonNum = parseFloat(lon);
            return (
                !isNaN(latNum) &&
                !isNaN(lonNum) &&
                latNum >= -90 &&
                latNum <= 90 &&
                lonNum >= -180 &&
                lonNum <= 180
            );
        }

        async function procesarError(res) {
            const errorData = await res.json().catch(() => ({}));
            return {
                status: res.status,
                statusText: res.statusText,
                serverError: errorData,
                url: res.url,
                message: `Error ${res.status}: ${res.statusText}`,
            };
        }

        obtenerUbicacionYEnviar();
        intervalo = setInterval(obtenerUbicacionYEnviar, 15000);
    });

    onDestroy(() => clearInterval(intervalo));
</script>

{#if error}
    <div class="error-box" role="alert">
        <h3>❌ {error.message}</h3>
        <p class="device-info">
            <strong>Dispositivo:</strong>
            {#if isMobile}
                Móvil 📱
            {:else}
                Escritorio 💻
            {/if}
        </p>

        {#if error.details.serverError}
            <div class="error-details">
                <strong>Detalles del servidor:</strong>
                <pre>{JSON.stringify(error.details.serverError, null, 2)}</pre>
            </div>
        {/if}
    </div>
{/if}

<div class="location-info" role="status">
    <h2>📍 Ubicación actual</h2>
    <p><strong>Latitud:</strong> {lat || "Obteniendo ubicación..."}</p>
    <p><strong>Longitud:</strong> {lon || "Obteniendo ubicación..."}</p>

    {#if lat && lon}
        <div class="status success">✅ Ubicación enviada correctamente</div>
    {:else}
        <div class="status loading">⌛ Enviando ubicación...</div>
    {/if}
</div>

<style>
    :global(body) {
        margin: 0;
        padding: 0;
        font-family: "Segoe UI", sans-serif;
        background-color: #f0f4f8;
    }

    .error-box {
        background-color: #ffebee;
        border-left: 5px solid #d32f2f;
        margin: 1rem auto;
        padding: 1rem;
        border-radius: 8px;
        max-width: 600px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        color: #b71c1c;
    }

    .error-box h3 {
        font-size: 1.3rem;
        margin-bottom: 0.5rem;
    }

    .device-info {
        font-size: 1rem;
    }

    .error-details {
        margin-top: 0.5rem;
        background-color: #ffcdd2;
        padding: 0.5rem;
        border-radius: 4px;
        overflow-x: auto;
    }

    .location-info {
        background-color: #ffffff;
        border-left: 5px solid #1976d2;
        margin: 1.5rem auto;
        padding: 1.2rem;
        border-radius: 10px;
        max-width: 600px;
        box-shadow: 0 2px 6px rgba(0, 0, 0, 0.08);
        color: #0d47a1;
        font-size: 1.1rem;
        text-align: center;
    }

    .location-info h2 {
        font-size: 1.5rem;
        margin-bottom: 1rem;
        color: #1565c0;
    }

    .location-info p {
        margin: 0.5rem 0;
    }

    .status {
        margin-top: 1rem;
        font-weight: bold;
        padding: 0.6rem;
        border-radius: 6px;
    }

    .status.success {
        background-color: #e8f5e9;
        color: #2e7d32;
        border: 2px solid #66bb6a;
    }

    .status.loading {
        background-color: #fffde7;
        color: #f9a825;
        border: 2px dashed #fdd835;
    }

    @media (max-width: 600px) {
        .error-box,
        .location-info {
            margin: 1rem;
            padding: 1rem;
        }

        .location-info h2 {
            font-size: 1.3rem;
        }

        .status {
            font-size: 1rem;
        }
    }
</style>
