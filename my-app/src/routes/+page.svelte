<script>
    import { onMount, onDestroy } from "svelte";
    import { Geolocation } from "@capacitor/geolocation";

    let lat = "";
    let lon = "";
    let intervalo = null;
    const discapacitado_id = 12;

    // Mueve toda la lógica de geolocalización aquí
    onMount(async () => {
        async function obtenerUbicacionYEnviar() {
            try {
                const position = await Geolocation.getCurrentPosition();
                lat = position.coords.latitude;
                lon = position.coords.longitude;

                console.log("Ubicación actual:", lat, lon);

                const res = await fetch(
                    `https://proyectomascotas.onrender.com/${discapacitado_id}`,
                    {
                        method: "PUT",
                        headers: {
                            "Content-Type": "application/json",
                        },
                        body: JSON.stringify({
                            latitud: lat,
                            longitud: lon,
                        }),
                    },
                );

                if (!res.ok) throw new Error("Error al actualizar ubicación");

                const data = await res.json();
                console.log("Respuesta del servidor:", data);
            } catch (error) {
                console.error("Error:", error);
            }
        }

        // Ejecuta inmediatamente y cada 15 segundos
        obtenerUbicacionYEnviar();
        intervalo = setInterval(obtenerUbicacionYEnviar, 15000);
    });

    onDestroy(() => {
        clearInterval(intervalo);
    });
</script>

<p>Latitud: {lat}</p>
<p>Longitud: {lon}</p>
