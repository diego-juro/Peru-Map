# 🗺️ Tutorial: Cómo hacer un mapa del Perú con resultados electorales y educación

En este tutorial aprenderás paso a paso cómo crear un mapa del Perú combinando:

- 🗳️ Partido político ganador por departamento  
- 🎓 Tasa de asistencia a educación superior (17–24 años)

---

## PASO 0: Instalar librerías

```text
options(timeout = 600)
options(download.file.method = "wininet")
install.packages("sf", repos = "https://cloud.r-project.org/")
install.packages("tidyverse")
```

```text
install.packages("purrr")

install.packages("ggrepel")

library(purrr)
library(ggplot2)
library(ggrepel)
library(sf)
library(tidyverse)
```

---

## PASO 1: Cargar el mapa del Perú

```text
peru_sf <- st_read("INEI_LIMITE_DEPARTAMENTAL.shp")
# Mapa base: PERU
ggplot(data = peru_sf) +
  geom_sf()
```

---

## PASO 2: Calcular centroides

```text
peru_sf <- peru_sf %>% mutate(centroid = map(geometry, st_centroid), 
                              coords = map(centroid, st_coordinates), 
                              coords_x = map_dbl(coords, 1), coords_y = map_dbl(coords, 2))


ggplot(data = peru_sf) +
  geom_sf(fill="orange", color="black", alpha = 0.7)+ 
  geom_text_repel(mapping = aes(coords_x, coords_y, label = NOMBDEP), size = 2)
```

---

## PASO 3: Cargar datos electorales

Fuente:  
https://github.com/jmcastagnetto/2021-elecciones-generales-peru-datos-de-onpe/blob/main/presidencial-resultados-partidos.csv

```text
data=read_csv('presidencial-resultados-partidos.csv')
```

---

## PASO 4: Obtener partido ganador por departamento

```text
ganadores_dep <- data %>%
  group_by(departamento, partido) %>%
  summarise(votos = sum(total_votos, na.rm = TRUE), .groups = "drop") %>%
  group_by(departamento) %>%
  slice_max(order_by = votos, n = 1, with_ties = FALSE) %>%
  ungroup()

ganadores_dep
```

---

## PASO 5: Crear dataset de educación

Fuente:  
https://www.inei.gob.pe/media/MenuRecursivo/publicaciones_digitales/Est/Lib1919/libro.pdf

```text
educacion <- tibble::tibble(
  departamento = c(
    "AREQUIPA","CUSCO","TACNA","LIMA METROPOLITANA","JUNIN","MOQUEGUA",
    "LIMA","PUNO","HUANCAVELICA","TUMBES","APURIMAC","CALLAO","PASCO",
    "ICA","ANCASH","AYACUCHO","LAMBAYEQUE","MADRE DE DIOS","AMAZONAS",
    "SAN MARTIN","LA LIBERTAD","HUANUCO","CAJAMARCA","UCAYALI","PIURA","LORETO"
  ),
  tasa_asistencia_superior = c(
    38.4, 37.3, 30.9, 30.7, 29.8, 29.8,
    29.4, 28.6, 28.2, 27.7, 27.7, 27.5, 26.8,
    26.7, 26.3, 25.5, 24.5, 24.1, 23.7,
    23.4, 22.5, 21.0, 20.2, 19.8, 16.7, 15.8
  )
)

educacion
```

---

## PASO 6: Resolver inconsistencia en Lima

```text
educacion_fix <- educacion %>%
  mutate(departamento = case_when(
    departamento == "LIMA METROPOLITANA" ~ "LIMA",
    TRUE ~ departamento
  ))

educacion_fix
```

---

## PASO 7: Unir las bases de datos

```text
peru_datos <- peru_sf %>%
  left_join(ganadores_dep, by = c("NOMBDEP" = "departamento")) %>%
  left_join(educacion_fix, by = c("NOMBDEP" = "departamento"))
```

---

## PASO 8: Crear variable categórica

```text
peru_datos <- peru_datos %>%
  mutate(
    educ_cat = cut(
      tasa_asistencia_superior,
      breaks = c(0, 20, 25, 30, 35, 40),
      labels = c("<20", "20–25", "25–30", "30–35", "35+"),
      include.lowest = TRUE
    )
  )
```

---

## PASO 9: Crear mapa final

```text
ggplot(peru_datos) +
  geom_sf(aes(fill = educ_cat, color = partido), size = 0.8) +
  scale_fill_brewer(palette = "YlGnBu") +
  scale_color_manual(values = c(
    "FUERZA POPULAR" = "orange",   # Poner nombre del partido 
    "PARTIDO POLITICO NACIONAL PERU LIBRE" = "red",
    "AVANZA PAIS - PARTIDO DE INTEGRACION SOCIAL" = "blue",
    "ALIANZA PARA EL PROGRESO" = "purple"
  )) +
  labs(
    title = "Educación superior y resultado electoral",
    subtitle = "Tasa de asistencia a educación superior (17–24 años)",
    fill = "Tasa (%)",
    color = "Partido ganador",
    caption = "Fuente: INEI (2022) +
resultados electorales oficiales"
  ) +
  theme_bw() +
  theme(
    plot.title = element_text(size = 18, face = "bold"),
    plot.subtitle = element_text(size = 12)
  )
```


