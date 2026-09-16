# Predicción de resultados de partidas de ajedrez

Proyecto de aprendizaje supervisado que utiliza posiciones de partidas de ajedrez para predecir el resultado final: victoria de blancas (`1-0`), victoria de negras (`0-1`) o tablas (`1/2-1/2`).

El flujo principal está en [decision_tree_chess_limpio.ipynb](decision_tree_chess_limpio.ipynb). El notebook [decision_tree_chess.ipynb](decision_tree_chess.ipynb) conserva el historial de exploración y pruebas del desarrollo.

## Estado actual

El notebook limpio realiza este flujo:

1. Lee partidas PGN comprimidas con Zstandard.
2. Selecciona la posición situada a 10 movimientos completos del final.
3. Calcula características de la posición y una evaluación de Stockfish.
4. Extrae 5.000 observaciones únicas y hace una división estratificada 80/20.
5. Ajusta un `DecisionTreeClassifier` con `GridSearchCV`.
6. Ajusta un `RandomForestClassifier` con un grid independiente.
7. Serializa ambos modelos con sus metadatos.

La semilla aleatoria es `42` y ambos grid searches utilizan `accuracy` con validación cruzada de 5 particiones. Stockfish se ejecuta a profundidad 8.

## Estructura

```text
.
├── DB/
│   ├── lichess_db_standard_rated_2013-01.pgn.zst
│   └── predicciones.pgn.zst
├── decision_tree_chess.ipynb
├── decision_tree_chess_limpio.ipynb
├── modelo_chess_random_forest.joblib
├── modelo_chess_arbol_y_random_forest.joblib  # generado al ejecutar el notebook limpio
└── stockfish/                 # local e ignorado por Git
```

La carpeta `stockfish/` no se versiona porque contiene el binario grande del motor. Debe existir localmente para ejecutar el notebook limpio. El binario probado es Stockfish 19 universal para macOS (`arm64` y `x86_64`), situado en `stockfish/stockfish`.

## Requisitos

- Python 3
- `python-chess`
- `joblib`
- `numpy`
- `pandas`
- `zstandard`
- `scikit-learn`
- Stockfish en `stockfish/stockfish`

## Funciones principales

### `partidas_en_stream(ruta)`

Genera partidas PGN una a una desde un archivo `.pgn.zst`, sin cargar toda la base de datos en memoria.

### `tiene_peon_pasado(tablero, color)`

Comprueba si un bando tiene un peón pasado: no hay peones rivales delante en su misma columna ni en las columnas adyacentes.

### `tiene_torre_adelantada(tablero, color)`

Indica si un bando tiene una torre en las dos filas más cercanas al lado rival.

### `movimientos_legales(tablero, color)`

Cuenta los movimientos legales disponibles para un color.

### `evaluacion_stockfish(tablero, motor)`

Consulta Stockfish y devuelve la evaluación en peones desde la perspectiva de las blancas. Los valores negativos favorecen a las negras.

### `estado_tablero(tablero, blancas_enrocadas, negras_enrocadas, motor)`

Construye las características de una posición: damas, enroques, pareja de alfiles, peones pasados, material, peones, movilidad, torres adelantadas y evaluación de Stockfish.

### `partida_a_dataframe(partida, movimientos_desde_final, motor)`

Reproduce una partida hasta la posición objetivo, calcula sus características y devuelve una fila de `pandas.DataFrame` con la variable objetivo `y`.

### `extrae_caracteristicas(ruta, numero_observaciones, movimientos_desde_final)`

Extrae observaciones únicas de varias partidas. Abre un único proceso de Stockfish, lo reutiliza durante la extracción y lo cierra al terminar.

## Características

`CARACTERISTICAS` define el orden que deben respetar los datos de entrenamiento y las predicciones:

```text
reina_blancas, reina_negras
blancas_enrocadas, negras_enrocadas
dos_alfiles_blancas, dos_alfiles_negras
peon_pasado_blancas, peon_pasado_negras
mas_piezas_blancas, mas_piezas_negras
mas_peones_blancas, mas_peones_negras
ventaja_material
movilidad_blancas, movilidad_negras
torre_adelantada_blancas, torre_adelantada_negras
evaluacion_stockfish
```

## Modelos serializados

Al ejecutar el notebook limpio se generan ambos modelos en `modelo_chess_arbol_y_random_forest.joblib`:

```python
{
	"arbol": arbol_optimizado,
	"bosque": bosque_aleatorio,
	"caracteristicas": CARACTERISTICAS,
	"resultado_a_clase": RESULTADO_A_CLASE,
	"stockfish_depth": STOCKFISH_DEPTH,
}
```

Para cargar el bosque:

```python
import joblib

paquete = joblib.load("modelo_chess_arbol_y_random_forest.joblib")
modelo = paquete["bosque"]
```

## Git y Stockfish

`stockfish/` está incluido en `.gitignore` porque el binario no debe subirse a GitHub. El ejecutable debe instalarse o copiarse localmente en esa ruta antes de ejecutar el notebook.