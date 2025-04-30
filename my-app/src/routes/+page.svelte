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
    <div class="error-box">
        <h3>{error.message}</h3>
        {#if isMobile}
            <p><strong>Dispositivo:</strong> Móvil 📱</p>
        {:else}
            <p><strong>Dispositivo:</strong> Escritorio 💻</p>
        {/if}
        {#if error.details.serverError}
            <pre>{JSON.stringify(error.details.serverError, null, 2)}</pre>
        {/if}
    </div>
{/if}

<p>Latitud: {lat || "Obteniendo..."}</p>
<p>Longitud: {lon || "Obteniendo..."}</p>

<style>
    .error-box {
        background: #fff3e0;
        border-left: 4px solid #ffa000;
        padding: 1rem;
        margin: 1rem 0;
    }
    pre {
        background: #f5f5f5;
        padding: 0.5rem;
        font-size: 0.8rem;
    }
</style>
