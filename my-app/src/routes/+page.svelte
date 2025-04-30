<script>
    import { onMount, onDestroy } from "svelte";
    import { Geolocation } from "@capacitor/geolocation";

    let lat = "";
    let lon = "";
    let intervalo = null;

    const discapacitado_id = 12; // Coloca el ID real aquí

    async function obtenerUbicacionYEnviar() {
        try {
            const position = await Geolocation.getCurrentPosition();
            lat = position.coords.latitude;
            lon = position.coords.longitude;

            console.log("Ubicación actual:", lat, lon);

            const res = await fetch(
                `https://tu-api.com/discapacitado/${discapacitado_id}/ubicacion`,
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

            if (!res.ok) {
                const errorData = await res.json();
                throw new Error(
                    errorData.detail || "Error al actualizar ubicación",
                );
            }

            const data = await res.json();
            console.log("Respuesta del servidor:", data);
        } catch (error) {
            console.error("Error al obtener o enviar ubicación:", error);
        }
    }

    onMount(() => {
        obtenerUbicacionYEnviar();
        intervalo = setInterval(obtenerUbicacionYEnviar, 15000); // cada 15 segundos
    });

    onDestroy(() => {
        clearInterval(intervalo);
    });
</script>

<p>Latitud: {lat}</p>
<p>Longitud: {lon}</p>
