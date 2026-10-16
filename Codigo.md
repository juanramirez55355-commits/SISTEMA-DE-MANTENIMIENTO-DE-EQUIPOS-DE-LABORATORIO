# SISTEMA PREVENTIVO DE MANTENIMIENTO DE EQUIPOS 
# ============================================================
# SISTEMA DE MANTENIMIENTO PREVENTIVO DE EQUIPOS 
# Lenguaje: Python
# Interfaz gráfica: Tkinter
# Base de datos: SQLite
#
# IMPORTANTE:
# Las líneas que empiezan con # son comentarios explicativos.
# Python no ejecuta los comentarios; sirven para entender el código.
# ============================================================


# ============================================================
# 1. IMPORTAR BIBLIOTECAS
# ============================================================

import sqlite3  # Permite guardar y consultar información en una base de datos SQLite.
import tkinter as tk  # Permite crear la ventana, etiquetas, campos y botones.
from tkinter import ttk, messagebox, simpledialog  # Ventanas de tablas, mensajes y solicitud de clave.

# Clave requerida para borrar permanentemente todos los equipos. Cámbiala por una propia.
CLAVE_BORRADO = "2468"

# NOTA: Se quitaron datetime y timedelta porque este programa no los utiliza.


# ============================================================
# 2. CONECTAR CON LA BASE DE DATOS
# ============================================================

# Nombre del archivo donde se guardarán los equipos.
# Puedes cambiarlo, pero se creará o utilizará otro archivo de base de datos.
conexion = sqlite3.connect("mantenimiento_laboratorio.db")

# El cursor permite ejecutar instrucciones SQL: INSERT, SELECT, UPDATE, etc.
cursor = conexion.cursor()

# Crear la tabla "equipos" solamente si todavía no existe.
# Cada fila de esta tabla representa un equipo registrado.
cursor.execute("""
CREATE TABLE IF NOT EXISTS equipos (
    id INTEGER PRIMARY KEY AUTOINCREMENT,  -- Identificador único que aumenta automáticamente.
    nombre TEXT NOT NULL,                  -- Nombre del equipo; no puede quedar vacío.
    horas_uso REAL NOT NULL,                -- Total de horas acumuladas de uso.
    intervalo REAL NOT NULL,                -- Horas que deben pasar entre mantenimientos.
    ultimo_mantenimiento REAL NOT NULL      -- Lectura del contador al hacer el último mantenimiento.
)
""")

# Guardar en el archivo los cambios hechos en la base de datos.
conexion.commit()


# ============================================================
# 3. REGISTRAR UN EQUIPO NUEVO
# ============================================================

def registrar_equipo():
    """Lee los campos, valida la información y guarda un equipo nuevo."""

    # Obtener el nombre escrito. strip() elimina espacios al principio y al final.
    nombre = entrada_nombre.get().strip()

    # Intentar convertir las horas y el intervalo a números decimales.
    # Si el usuario escribe letras, se mostrará un error.
    try:
        horas = float(entrada_horas.get())
        intervalo = float(entrada_intervalo.get())
    except ValueError:
        messagebox.showerror(
            "Error",
            "Las horas y el intervalo deben ser números."
        )
        return  # Detener la función si los datos no son válidos.

    # Verificar que el nombre no esté vacío.
    if not nombre:
        messagebox.showerror("Error", "Ingrese el nombre del equipo.")
        return

    # Las horas pueden ser cero, pero no negativas.
    # El intervalo debe ser mayor que cero.
    if horas < 0 or intervalo <= 0:
        messagebox.showerror(
            "Error",
            "Las horas no pueden ser negativas y el intervalo debe ser mayor que 0."
        )
        return

    # Evitar registrar dos equipos con el mismo nombre.
    # La comparación ignora mayúsculas/minúsculas y espacios en los extremos.
    cursor.execute("""
        SELECT id FROM equipos
        WHERE LOWER(TRIM(nombre)) = LOWER(TRIM(?))
    """, (nombre,))
    if cursor.fetchone():
        messagebox.showerror(
            "Equipo duplicado",
            f"Ya existe un equipo llamado '{nombre}'. Use otro nombre."
        )
        return

    # Insertar el nuevo equipo en la base de datos.
    # Los signos ? se reemplazan de forma segura por los valores de la tupla.
    # Se toma la lectura actual como punto de partida del mantenimiento.
    cursor.execute("""
        INSERT INTO equipos
        (nombre, horas_uso, intervalo, ultimo_mantenimiento)
        VALUES (?, ?, ?, ?)
    """, (nombre, horas, intervalo, horas))

    conexion.commit()  # Guardar el registro.

    limpiar_campos()  # Vaciar los campos del formulario.
    mostrar_equipos()  # Volver a cargar la tabla para mostrar el nuevo equipo.

    # Mostrar una confirmación al usuario.
    messagebox.showinfo(
        "Éxito",
        f"El equipo '{nombre}' fue registrado correctamente."
    )


# ============================================================
# 4. SUMAR HORAS DE USO A UN EQUIPO
# ============================================================

def actualizar_horas():
    """Suma horas al equipo seleccionado en la tabla."""

    # Obtener la fila que el usuario seleccionó en la tabla.
    equipo_seleccionado = tabla.selection()

    # Si no seleccionó ninguna fila, avisar y detener la función.
    if not equipo_seleccionado:
        messagebox.showwarning("Advertencia", "Seleccione un equipo.")
        return

    # Convertir a número las horas que se van a sumar.
    try:
        horas_nuevas = float(entrada_sumar_horas.get())
    except ValueError:
        messagebox.showerror(
            "Error",
            "Ingrese una cantidad válida de horas."
        )
        return

    # No permitir sumar cero ni cantidades negativas.
    if horas_nuevas <= 0:
        messagebox.showerror(
            "Error",
            "Las horas deben ser mayores que 0."
        )
        return

    # Obtener los datos de la fila seleccionada.
    item = tabla.item(equipo_seleccionado[0])

    # La primera columna de la tabla contiene el ID del equipo.
    equipo_id = item["values"][0]

    # Sumar las nuevas horas al total guardado para ese ID.
    cursor.execute("""
        UPDATE equipos
        SET horas_uso = horas_uso + ?
        WHERE id = ?
    """, (horas_nuevas, equipo_id))

    conexion.commit()  # Guardar el nuevo total de horas.

    # Limpiar el campo donde se escribieron las horas.
    entrada_sumar_horas.delete(0, tk.END)

    mostrar_equipos()  # Actualizar la tabla y los estados.

    messagebox.showinfo(
        "Actualizado",
        "Las horas de uso fueron actualizadas."
    )


# ============================================================
# 5. REGISTRAR QUE SE HIZO EL MANTENIMIENTO
# ============================================================

def realizar_mantenimiento():
    """Guarda las horas actuales como la lectura del último mantenimiento."""

    equipo_seleccionado = tabla.selection()

    # Es obligatorio seleccionar un equipo.
    if not equipo_seleccionado:
        messagebox.showwarning("Advertencia", "Seleccione un equipo.")
        return

    # Obtener el ID de la fila seleccionada.
    item = tabla.item(equipo_seleccionado[0])
    equipo_id = item["values"][0]

    # Consultar las horas actuales de ese equipo.
    cursor.execute("""
        SELECT horas_uso
        FROM equipos
        WHERE id = ?
    """, (equipo_id,))

    resultado = cursor.fetchone()  # Obtener el resultado de la consulta.

    if resultado:
        horas_actuales = resultado[0]

        # Registrar la lectura actual como el último mantenimiento.
        # El siguiente mantenimiento será esta lectura + el intervalo.
        cursor.execute("""
            UPDATE equipos
            SET ultimo_mantenimiento = ?
            WHERE id = ?
        """, (horas_actuales, equipo_id))

        conexion.commit()  # Guardar el mantenimiento.

        mostrar_equipos()  # Actualizar la tabla y el estado del equipo.

        messagebox.showinfo(
            "Mantenimiento",
            "El mantenimiento fue registrado correctamente."
        )


# ============================================================
# 6. CALCULAR EL ESTADO DEL EQUIPO
# ============================================================

def obtener_estado(horas_uso, intervalo, ultimo_mantenimiento):
    """
    Devuelve uno de estos estados:
    - VENCIDO: se alcanzó o superó la lectura prevista.
    - PRÓXIMO: falta el 20 % o menos del intervalo.
    - AL DÍA: falta más del 20 % del intervalo.
    """

    # Calcular a qué lectura del contador toca el siguiente mantenimiento.
    siguiente = ultimo_mantenimiento + intervalo

    # Calcular cuántas horas faltan.
    diferencia = siguiente - horas_uso

    # Si faltan cero horas o un valor negativo, el mantenimiento está vencido.
    if diferencia <= 0:
        return "VENCIDO"

    # CAMBIO PERSONALIZABLE:
    # 0.20 significa 20 % del intervalo.
    # Por ejemplo, cambia 0.20 por 0.30 para avisar cuando falte el 30 %.
    if diferencia <= intervalo * 0.20:
        return "PRÓXIMO"

    # Si no cumple las condiciones anteriores, está al día.
    return "AL DÍA"


# ============================================================
# 7. MOSTRAR LOS EQUIPOS EN LA TABLA
# ============================================================

def mostrar_equipos():
    """Consulta la base de datos y actualiza la tabla de la ventana."""

    # Borrar de la pantalla las filas actuales antes de volver a cargarlas.
    # Esto no borra los registros de la base de datos.
    for fila in tabla.get_children():
        tabla.delete(fila)

    # Consultar los equipos y ordenarlos alfabéticamente por nombre.
    cursor.execute("""
        SELECT id, nombre, horas_uso, intervalo, ultimo_mantenimiento
        FROM equipos
        ORDER BY nombre
    """)

    equipos = cursor.fetchall()  # Obtener todos los equipos encontrados.

    # Procesar cada equipo de la lista.
    for equipo in equipos:
        equipo_id, nombre, horas, intervalo, ultimo = equipo

        # Calcular la lectura a la que corresponde el próximo mantenimiento.
        proximo = ultimo + intervalo

        # Determinar si está vencido, próximo o al día.
        estado = obtener_estado(horas, intervalo, ultimo)

        # Mostrar los números con un decimal, por ejemplo 125.0.
        texto_proximo = f"{proximo:.1f} h"

        # Añadir una fila a la tabla.
        tabla.insert(
            "",
            tk.END,
            values=(
                equipo_id,
                nombre,
                f"{horas:.1f}",
                f"{intervalo:.1f}",
                texto_proximo,
                estado
            )
        )

    # Aplicar los colores de acuerdo con el estado de cada fila.
    actualizar_colores()


# ============================================================
# 8. COLOREAR LAS FILAS SEGÚN EL ESTADO
# ============================================================

def actualizar_colores():
    """Asigna una etiqueta de color a cada fila de la tabla."""

    for item in tabla.get_children():
        # Obtener los valores de la fila.
        valores = tabla.item(item)["values"]

        # La posición 5 corresponde a la sexta columna: Estado.
        estado = valores[5]

        # Asignar una etiqueta según el estado.
        if estado == "VENCIDO":
            tabla.item(item, tags=("vencido",))
        elif estado == "PRÓXIMO":
            tabla.item(item, tags=("proximo",))
        else:
            tabla.item(item, tags=("aldia",))


# ============================================================
# 9. MOSTRAR ALERTAS DE MANTENIMIENTO
# ============================================================

def mostrar_alertas():
    """Muestra una ventana con equipos vencidos y próximos."""

    # Consultar los datos necesarios para calcular los estados.
    cursor.execute("""
        SELECT nombre, horas_uso, intervalo, ultimo_mantenimiento
        FROM equipos
    """)

    equipos = cursor.fetchall()

    # Listas que guardarán los nombres y mensajes de cada categoría.
    vencidos = []
    proximos = []

    # Revisar cada equipo.
    for nombre, horas, intervalo, ultimo in equipos:
        estado = obtener_estado(horas, intervalo, ultimo)

        if estado == "VENCIDO":
            vencidos.append(f"- {nombre}: mantenimiento vencido")

        elif estado == "PRÓXIMO":
            siguiente = ultimo + intervalo
            faltan = siguiente - horas
            proximos.append(f"- {nombre}: faltan {faltan:.1f} horas")

    # Empezar con un mensaje vacío e ir agregando las alertas encontradas.
    mensaje = ""

    if vencidos:
        mensaje += "MANTENIMIENTO VENCIDO\n\n"
        mensaje += "\n".join(vencidos)
        mensaje += "\n\n"

    if proximos:
        mensaje += "MANTENIMIENTO PRÓXIMO\n\n"
        mensaje += "\n".join(proximos)

    # Si ambas listas están vacías, no hay alertas.
    if not mensaje:
        mensaje = "No hay equipos vencidos o próximos a mantenimiento."

    # Mostrar el resumen en una ventana emergente.
    messagebox.showinfo("Alertas de mantenimiento", mensaje)


# ============================================================
# 10. GENERAR EL REPORTE SEMANAL
# ============================================================

def generar_reporte():
    """
    Muestra equipos cuyo mantenimiento está a 56 horas de uso o menos.

    SUPOSICIÓN PERSONALIZABLE:
    Se estiman 8 horas de uso al día durante 7 días.
    Este cálculo es por horas de uso, no por fechas de calendario.
    """

    # Consultar los datos de todos los equipos.
    cursor.execute("""
        SELECT nombre, horas_uso, intervalo, ultimo_mantenimiento
        FROM equipos
    """)

    equipos = cursor.fetchall()
    equipos_reporte = []

    # CAMBIAR ESTOS VALORES SI TU LABORATORIO USA OTRO HORARIO:
    HORAS_USO_DIARIAS = 8  # Estimación de horas de funcionamiento por día.
    DIAS_REPORTE = 7       # Cantidad de días que se quiere estimar.

    # Calcular el límite de horas para incluir un equipo en el reporte.
    HORAS_SEMANA = HORAS_USO_DIARIAS * DIAS_REPORTE

    for nombre, horas, intervalo, ultimo in equipos:
        proximo = ultimo + intervalo
        faltan = proximo - horas

        # Incluir equipos vencidos y equipos que necesitan mantenimiento
        # dentro del límite estimado de horas de uso.
        if faltan <= HORAS_SEMANA:
            equipos_reporte.append((nombre, horas, proximo, faltan))

    # Crear una segunda ventana para el reporte.
    ventana = tk.Toplevel(root)
    ventana.title("Reporte semanal")  # CAMBIAR: título de la ventana.
    ventana.geometry("650x400")       # CAMBIAR: ancho x alto en píxeles.

    # Título que aparece dentro de la ventana.
    titulo_reporte = tk.Label(
        ventana,
        text="REPORTE DE MANTENIMIENTO - PRÓXIMOS 7 DÍAS",
        font=("Arial", 14, "bold")
    )
    titulo_reporte.pack(pady=15)

    # Si no hay equipos para mostrar, presentar un mensaje y terminar.
    if not equipos_reporte:
        tk.Label(
            ventana,
            text="No hay equipos que necesiten mantenimiento esta semana.",
            font=("Arial", 11)
        ).pack(pady=30)
        return

    # Crear un área de texto para mostrar los detalles del reporte.
    texto = tk.Text(ventana, width=75, height=15)
    texto.pack(padx=20, pady=10)

    # Escribir el encabezado del reporte.
    texto.insert(tk.END, "EQUIPOS QUE NECESITAN MANTENIMIENTO\n")
    texto.insert(tk.END, "=" * 65 + "\n\n")

    # Escribir los datos de cada equipo incluido en el reporte.
    for nombre, horas, proximo, faltan in equipos_reporte:
        if faltan <= 0:
            estado = "VENCIDO"
        else:
            estado = f"Faltan {faltan:.1f} horas"

        texto.insert(
            tk.END,
            f"Equipo: {nombre}\n"
            f"Horas actuales: {horas:.1f}\n"
            f"Mantenimiento a las: {proximo:.1f} horas\n"
            f"Estado: {estado}\n"
            f"{'-' * 65}\n"
        )

    # Evitar que el usuario edite accidentalmente el texto del reporte.
    texto.config(state=tk.DISABLED)


# ============================================================
# 11. LIMPIAR LOS CAMPOS DEL FORMULARIO
# ============================================================

def limpiar_campos():
    """Borra el contenido de los tres campos para registrar equipos."""

    entrada_nombre.delete(0, tk.END)     # Borra el nombre.
    entrada_horas.delete(0, tk.END)      # Borra las horas.
    entrada_intervalo.delete(0, tk.END)  # Borra el intervalo.


# ============================================================
# 12. CREAR LA VENTANA PRINCIPAL
# ============================================================

root = tk.Tk()

# CAMBIAR: texto que aparece en la barra superior de la ventana.
root.title("Sistema de Mantenimiento - Laboratorio")

# CAMBIAR: tamaño de la ventana, en formato "ANCHOxALTO".
root.geometry("950x650")

# False, False significa que no se permite cambiar el tamaño manualmente.
# Puedes usar True, True si quieres permitir cambiar ancho y alto.
root.resizable(False, False)


# ============================================================
# 13. TÍTULO Y SUBTÍTULO
# ============================================================

# Etiqueta principal de la ventana.
titulo = tk.Label(
    root,
    text="SISTEMA DE MANTENIMIENTO DE EQUIPOS",  # CAMBIAR: texto del título.
    font=("Arial", 20, "bold")                   # CAMBIAR: fuente y tamaño.
)
titulo.pack(pady=15)  # Espacio vertical alrededor del título.

# Texto que aparece debajo del título.
subtitulo = tk.Label(
    root,
    text="Control de horas de uso y mantenimiento preventivo",
    font=("Arial", 11)
)
subtitulo.pack()


# ============================================================
# 14. FORMULARIO PARA REGISTRAR EQUIPOS
# ============================================================

# Recuadro que agrupa los campos de registro.
frame_registro = tk.LabelFrame(
    root,
    text="Registrar equipo",  # Texto del borde del recuadro.
    font=("Arial", 11, "bold"),
    padx=15,                   # Espacio horizontal interior.
    pady=15                    # Espacio vertical interior.
)
frame_registro.pack(padx=20, pady=15, fill="x")


# Etiqueta y campo para el nombre del equipo.
tk.Label(frame_registro, text="Nombre del equipo:").grid(
    row=0, column=0, padx=5, pady=5
)
entrada_nombre = tk.Entry(frame_registro, width=25)  # CAMBIAR: ancho del campo.
entrada_nombre.grid(row=0, column=1, padx=5)


# Etiqueta y campo para las horas actuales.
tk.Label(frame_registro, text="Horas actuales:").grid(
    row=0, column=2, padx=5
)
entrada_horas = tk.Entry(frame_registro, width=15)
entrada_horas.grid(row=0, column=3, padx=5)


# Etiqueta y campo para el intervalo de mantenimiento.
tk.Label(frame_registro, text="Intervalo (horas):").grid(
    row=0, column=4, padx=5
)
entrada_intervalo = tk.Entry(frame_registro, width=15)
entrada_intervalo.grid(row=0, column=5, padx=5)


# Botón que ejecuta registrar_equipo() al pulsarlo.
boton_registrar = tk.Button(
    frame_registro,
    text="Registrar equipo",        # CAMBIAR: texto del botón.
    command=registrar_equipo,       # Función que se ejecuta al hacer clic.
    bg="#2E7D32",                   # CAMBIAR: color de fondo en hexadecimal.
    fg="white",                     # Color del texto.
    padx=10
)
boton_registrar.grid(row=0, column=6, padx=10)


# ============================================================
# 15. TABLA DE EQUIPOS REGISTRADOS
# ============================================================

# Recuadro que contiene la tabla.
frame_tabla = tk.LabelFrame(
    root,
    text="Equipos registrados",
    font=("Arial", 11, "bold"),
    padx=10,
    pady=10
)
frame_tabla.pack(padx=20, pady=5, fill="both", expand=True)

# Nombres y orden de las columnas de la tabla.
# Si cambias estas columnas, también debes revisar los valores que inserta mostrar_equipos().
columnas = (
    "ID",
    "Equipo",
    "Horas actuales",
    "Intervalo",
    "Próximo mantenimiento",
    "Estado"
)

# Crear la tabla.
tabla = ttk.Treeview(
    frame_tabla,
    columns=columnas,
    show="headings",  # Mostrar encabezados de columnas.
    height=12         # CAMBIAR: cantidad aproximada de filas visibles.
)

# Recorrer las columnas para configurar encabezado, ancho y alineación.
for columna in columnas:
    tabla.heading(columna, text=columna)
    tabla.column(columna, width=140, anchor="center")

# La columna ID es más estrecha que las demás.
tabla.column("ID", width=50)

# Mostrar la tabla y permitirle ocupar el espacio disponible del recuadro.
tabla.pack(fill="both", expand=True)


# ============================================================
# 16. COLORES DE LOS ESTADOS EN LA TABLA
# ============================================================

# CAMBIAR: los colores se indican con códigos hexadecimales.
# Fondo rojo claro y texto rojo oscuro para equipos vencidos.
tabla.tag_configure(
    "vencido",
    background="#ffcdd2",
    foreground="#b71c1c"
)

# Fondo amarillo claro para equipos próximos a mantenimiento.
tabla.tag_configure(
    "proximo",
    background="#fff9c4",
    foreground="#795548"
)

# Fondo verde claro para equipos al día.
tabla.tag_configure(
    "aldia",
    background="#c8e6c9",
    foreground="#1b5e20"
)


# ============================================================
# 17. FORMULARIO PARA SUMAR HORAS Y REGISTRAR MANTENIMIENTO
# ============================================================

frame_horas = tk.LabelFrame(
    root,
    text="Actualizar horas de uso",
    font=("Arial", 11, "bold"),
    padx=10,
    pady=10
)
frame_horas.pack(padx=20, pady=10, fill="x")

# Etiqueta que explica qué debe escribir el usuario.
tk.Label(frame_horas, text="Horas utilizadas:").pack(side="left", padx=10)

# Campo donde se escribe la cantidad de horas que se va a sumar.
entrada_sumar_horas = tk.Entry(frame_horas, width=15)
entrada_sumar_horas.pack(side="left", padx=10)

# Botón para sumar horas al equipo seleccionado.
tk.Button(
    frame_horas,
    text="Sumar horas",
    command=actualizar_horas,
    bg="#1565C0",
    fg="white"
).pack(side="left", padx=10)

# Botón para registrar el mantenimiento del equipo seleccionado.
tk.Button(
    frame_horas,
    text="Registrar mantenimiento",
    command=realizar_mantenimiento,
    bg="#6A1B9A",
    fg="white"
).pack(side="left", padx=10)


def borrar_todos_los_equipos():
    """Borra permanentemente todos los equipos solo si la clave es correcta."""

    clave = simpledialog.askstring(
        "Clave requerida",
        "Ingrese la clave para borrar TODOS los equipos:",
        show="*",
        parent=root
    )

    # Si el usuario cancela o deja el campo vacío, no borrar nada.
    if clave is None:
        return

    if clave != CLAVE_BORRADO:
        messagebox.showerror("Clave incorrecta", "No se borró ningún equipo.")
        return

    confirmar = messagebox.askyesno(
        "Confirmar borrado",
        "Esto eliminará permanentemente todos los equipos guardados. ¿Desea continuar?",
        parent=root
    )
    if not confirmar:
        return

    cursor.execute("DELETE FROM equipos")
    conexion.commit()
    mostrar_equipos()  # Vuelve a consultar la base de datos, ahora vacía.
    messagebox.showinfo("Lista borrada", "Se eliminaron todos los equipos guardados.")


# ============================================================
# 18. BOTONES INFERIORES
# ============================================================

# Frame es un contenedor sencillo para agrupar botones.
frame_botones = tk.Frame(root)
frame_botones.pack(pady=15)



# Botón para borrar de forma permanente, protegido con clave.
tk.Button(
    frame_botones,
    text="Borrar lista (clave)",
    command=borrar_todos_los_equipos,
    width=20,
    bg="#C62828",
    fg="white"
).grid(row=0, column=3, padx=10)

# Botón para mostrar las alertas de mantenimiento.
tk.Button(
    frame_botones,
    text="Ver alertas",
    command=mostrar_alertas,
    width=20,
    bg="#F9A825"
).grid(row=0, column=1, padx=10)

# Botón para abrir el reporte semanal estimado.
tk.Button(
    frame_botones,
    text="Reporte semanal",
    command=generar_reporte,
    width=20,
    bg="#00838F",
    fg="white"
).grid(row=0, column=2, padx=10)


# ============================================================
# 19. INICIAR EL PROGRAMA
# ============================================================

# Cargar en la tabla los equipos que ya están guardados en SQLite.
mostrar_equipos()

# Iniciar el ciclo de eventos de Tkinter.
# Mantiene la ventana abierta y permite responder a clics y teclado.
root.mainloop()

# Cuando se cierre la ventana principal, cerrar la conexión con la base de datos.
conexion.close()
