import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import pandas as pd
import numpy as np
import random
import simpy

# ==========================
# LISTA DE JUGADORES
# ==========================
jugadores = []

# ==========================
# CRECIMIENTO POR EDAD
# ==========================
def factor_crecimiento_por_edad(edad):
    """
    Jugadores jóvenes mejoran más rápido que jugadores mayores.
    """
    base = max(0.01, (22 - edad) * 0.03)
    return base

# ==========================
# SIMPY: CRECIMIENTO MES A MES
# ==========================
def proceso_jugador(env, datos_jugador, meses, resultados):
    """
    Simula crecimiento del jugador mes a mes usando SimPy.
    """
    edad = datos_jugador["edad"]
    goles = datos_jugador["goles_90"]
    asist = datos_jugador["asist_90"]
    pase = datos_jugador["pase_exitoso"]
    dist = datos_jugador["distancia_km"]

    crecimiento = factor_crecimiento_por_edad(edad)

    for _ in range(meses):
        yield env.timeout(1)  # 1 mes

        # Crecimientos pequeños, realistas
        goles += np.random.normal(crecimiento * 0.06, 0.01)
        asist += np.random.normal(crecimiento * 0.05, 0.01)
        pase += np.random.normal(crecimiento * 0.04, 0.01)
        dist += np.random.normal(crecimiento * 0.2, 0.05)

    resultados.append({
        "nombre": datos_jugador["nombre"],
        "edad": edad,
        "periodo_meses": meses,
        "goles_90": max(goles, 0),
        "asist_90": max(asist, 0),
        "pase_exitoso": min(max(pase, 0), 1),
        "distancia_km": max(dist, 0)
    })


def simular_todos_sin_interfaz(meses):
    """
    Ejecuta SimPy para todos los jugadores al mismo tiempo.
    """
    env = simpy.Environment()
    resultados = []

    for jugador in jugadores:
        env.process(proceso_jugador(env, jugador, meses, resultados))

    env.run()
    return resultados

# ==========================
# MONTE CARLO
# ==========================
def simular_monte_carlo(jugador, num_simulaciones=500):
    """
    Simula variabilidad del rendimiento del jugador.
    """
    goles = []
    asist = []
    pase = []
    dist = []

    edad = jugador["edad"]
    crecimiento = factor_crecimiento_por_edad(edad)

    for _ in range(num_simulaciones):
        goles.append(np.random.normal(jugador["goles_90"] + crecimiento * 0.1, 0.03))
        asist.append(np.random.normal(jugador["asist_90"] + crecimiento * 0.1, 0.03))
        pase.append(min(max(np.random.normal(jugador["pase_exitoso"], 0.02), 0), 1))
        dist.append(np.random.normal(jugador["distancia_km"] + crecimiento * 0.5, 0.2))

    return {
        "goles_prom": np.mean(goles),
        "asist_prom": np.mean(asist),
        "pase_prom": np.mean(pase),
        "dist_prom": np.mean(dist),
        "score": (
            np.mean(goles) * 0.35 +
            np.mean(asist) * 0.25 +
            np.mean(pase) * 0.20 +
            np.mean(dist) * 0.20
        )
    }

# ==========================
# SISTEMA DE RECOMENDACIONES
# ==========================
def recomendar_jugador(df, objetivo):
    if objetivo == "Goles":
        return df.loc[df["goles_90"].idxmax()]
    elif objetivo == "Asistencias":
        return df.loc[df["asist_90"].idxmax()]
    elif objetivo == "Pases Exitosos":
        return df.loc[df["pase_exitoso"].idxmax()]
    elif objetivo == "Intensidad":
        return df.loc[df["distancia_km"].idxmax()]
    else:
        return df.loc[df["score"].idxmax()]

# ==========================
# INTERFAZ GRÁFICA
# ==========================
ventana = tk.Tk()
ventana.title("Analizador y Simulador de Jugadores")
ventana.geometry("900x620")
ventana.configure(bg="#1e1e1e")

# --------------------------
# FORMULARIO
# --------------------------
frame_form = tk.LabelFrame(ventana, text="Datos del Jugador", fg="white", bg="#1e1e1e")
frame_form.pack(fill="x", padx=10, pady=5)

labels = ["Nombre", "Edad", "Goles/90", "Asistencias/90", "Pase Exitoso (0-1)", "Distancia km"]
entries = {}

for idx, label in enumerate(labels):
    tk.Label(frame_form, text=label, fg="white", bg="#1e1e1e").grid(row=idx, column=0, padx=5, pady=5)
    entry = tk.Entry(frame_form)
    entry.grid(row=idx, column=1, padx=5, pady=5)
    entries[label] = entry

def agregar_jugador():
    try:
        jugadores.append({
            "nombre": entries["Nombre"].get(),
            "edad": int(entries["Edad"].get()),
            "goles_90": float(entries["Goles/90"].get()),
            "asist_90": float(entries["Asistencias/90"].get()),
            "pase_exitoso": float(entries["Pase Exitoso (0-1)"].get()),
            "distancia_km": float(entries["Distancia km"].get()),
        })
        messagebox.showinfo("OK", "Jugador agregado correctamente.")
    except:
        messagebox.showerror("Error", "Datos inválidos.")

btn_agregar = tk.Button(frame_form, text="Agregar Jugador", command=agregar_jugador)
btn_agregar.grid(row=len(labels), column=0, columnspan=2, pady=10)

# --------------------------
# SIMULACIÓN
# --------------------------
frame_sim = tk.LabelFrame(ventana, text="Simulación", fg="white", bg="#1e1e1e")
frame_sim.pack(fill="x", padx=10, pady=5)

periodo_var = tk.StringVar(value="12")

tk.Label(frame_sim, text="Periodo (meses):", fg="white", bg="#1e1e1e").grid(row=0, column=0)
ttk.Combobox(frame_sim, textvariable=periodo_var, values=["6", "12", "24"]).grid(row=0, column=1)

# Tabla resultados
frame_tabla = tk.Frame(ventana, bg="#1e1e1e")
frame_tabla.pack(fill="both", expand=True)

tabla = ttk.Treeview(frame_tabla, columns=("nombre", "g", "a", "p", "d", "score"), show="headings")
columnas = ["Jugador", "Goles/90", "Asist/90", "Pase Exito", "Distancia", "Score"]
for col in range(len(columnas)):
    tabla.heading(col, text=columnas[col])
tabla.pack(fill="both", expand=True)

def ejecutar_simulacion():
    meses = int(periodo_var.get())
    resultados = simular_todos_sin_interfaz(meses)

    filas_montecarlo = []
    tabla.delete(*tabla.get_children())

    for jugador in resultados:
        mc = simular_monte_carlo(jugador)
        jugador_final = {
            **jugador,
            "score": mc["score"]
        }
        filas_montecarlo.append(jugador_final)

        tabla.insert("", "end", values=(
            jugador["nombre"],
            round(jugador["goles_90"], 3),
            round(jugador["asist_90"], 3),
            round(jugador["pase_exitoso"], 3),
            round(jugador["distancia_km"], 3),
            round(jugador_final["score"], 3)
        ))

    global df_resultados
    df_resultados = pd.DataFrame(filas_montecarlo)

btn_simular = tk.Button(frame_sim, text="Ejecutar Simulación", command=ejecutar_simulacion)
btn_simular.grid(row=0, column=2, padx=10)

# --------------------------
# RECOMENDACIONES
# --------------------------
frame_reco = tk.LabelFrame(ventana, text="Recomendación", fg="white", bg="#1e1e1e")
frame_reco.pack(fill="x", padx=10, pady=5)

criterio = tk.StringVar(value="Mejor General")
ttk.Combobox(
    frame_reco, textvariable=criterio,
    values=["Mejor General", "Goles", "Asistencias", "Pases Exitosos", "Intensidad"]
).grid(row=0, column=0)

def recomendar():
    if df_resultados.empty:
        messagebox.showerror("Error", "Primero ejecute la simulación.")
        return

    reco = recomendar_jugador(df_resultados, criterio.get())
    messagebox.showinfo("Recomendación", f"El mejor jugador es:\n{reco['nombre']}")

btn_reco = tk.Button(frame_reco, text="Recomendar", command=recomendar)
btn_reco.grid(row=0, column=1, padx=10)

ventana.mainloop()
