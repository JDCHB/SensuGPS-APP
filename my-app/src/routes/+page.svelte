<script>
    import { onMount, onDestroy } from "svelte";
    import { Geolocation } from "@capacitor/geolocation";

    let lat = "";
    let lon = "";
    let error = null; // Cambiado a objeto para más detalles
    let intervalo = null;
    const discapacitado_id = 12;

    onMount(async () => {
        async function obtenerUbicacionYEnviar() {
            try {
                const position = await Geolocation.getCurrentPosition();
                lat = position.coords.latitude.toFixed(6);
                lon = position.coords.longitude.toFixed(6);

                // Validación de coordenadas
                if (isNaN(lat) || isNaN(lon)) {
                    throw {
                        message: "Coordenadas inválidas",
                        details: { lat, lon },
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
                            latitud: parseFloat(lat),
                            longitud: parseFloat(lon),
                        }),
                    },
                );

                if (!res.ok) {
                    const errorData = await res.json().catch(() => ({}));
                    throw {
                        status: res.status,
                        statusText: res.statusText,
                        serverError: errorData,
                        url: res.url,
                    };
                }

                const data = await res.json();
                console.log("Respuesta exitosa:", data);
                error = null; // Limpiar errores previos
            } catch (err) {
                error = {
                    message: err.message || "Error desconocido",
                    details: err,
                };
                console.error("Error completo:", JSON.stringify(err, null, 2));
            }
        }

        obtenerUbicacionYEnviar();
        intervalo = setInterval(obtenerUbicacionYEnviar, 5000);
    });

    onDestroy(() => clearInterval(intervalo));
</script>

{#if error}
    <div class="error-box">
        <h3>Error al actualizar ubicación</h3>
        <p><strong>Mensaje:</strong> {error.message}</p>

        {#if error.details.status}
            <p><strong>Código HTTP:</strong> {error.details.status}</p>
            <p><strong>Respuesta del servidor:</strong></p>
            <pre>{JSON.stringify(error.details.serverError, null, 2)}</pre>
        {/if}
    </div>
{/if}

<p>Latitud: {lat || "No disponible"}</p>
<p>Longitud: {lon || "No disponible"}</p>

<style>
    .error-box {
        background: #ffebee;
        border: 1px solid #f44336;
        border-radius: 4px;
        padding: 1rem;
        margin-bottom: 1rem;
        color: #b71c1c;
    }
    pre {
        background: #f5f5f5;
        padding: 0.5rem;
        overflow-x: auto;
    }
</style>
