# P3 – FSM Expendedora de pastelitos: Timing y HDL

Máquina expendedora de pastelitos dividida en dos máquinas de estados que trabajan juntas:

- **Mealy – Crédito y venta:** guarda el crédito (0 $ a 3 $) y decide si una compra procede (`V`), qué tipo se entrega (`Tp`) o si hay error por crédito insuficiente (`E`).
- **Moore – Mecanismo de entrega:** gira el resorte del producto, espera a que caiga el pastelito, abre la puerta y avisa que está ocupada (`Busy`).

`V` y `Tp` van de la Mealy a la Moore, y `Busy` regresa a la Mealy como `B` para no cobrar mientras se está entregando.

---

## 🎥 Videos

| Inciso | Contenido | Video |
|---|---|---|
| **1. Timing** | Tramo de FF a FF entre la Moore y la Mealy con componentes reales (Digikey / Texas Instruments): restricción de setup, frecuencia máxima real (Fc) y restricción de hold | [Ver video](https://youtu.be/LpnqJdzzr94) |
| **2. HDL** | Migración a SystemVerilog (top / mealy / moore), simulación con testbench en Vivado y discusión de RTL Analysis, Synthesis e Implementation | [Ver video](https://youtu.be/T1fyBmXN4ZQ) |
| **2d. Basys3** | Demostración de la FSM programada en la tarjeta Basys3 | [Ver video](https://youtu.be/FXuJcfkWOdQ) |

---

## 1. Timing

**Tramo elegido (Moore → Mealy):** FF `S1` de la Moore → NOR (`~B = NOR(S1, S0)`) → AND de 5 entradas en 3 niveles (término `S1·~S0·K·~T·~B`) → OR de 6 términos en 3 niveles → entrada `D` del FF `S0` de la Mealy.

| Pieza | Función | t_pd mín | t_pd máx |
|---|---|---|---|
| SN74LVC1G175 | Flip-flop D | t_clk→Q 1.6 ns | t_clk→Q 5.7 ns · t_setup 2.0 ns · t_hold 0.5 ns |
| SN74LVC1G02 | NOR (~B) | 1.0 ns | 4.5 ns |
| SN74LVC1G08 | AND de 2 entradas | 1.0 ns | 4.5 ns |
| SN74AC32 | OR de 2 entradas | 1.0 ns | 11 ns |

*Condiciones: VCC = 3.3 V, CL = 30–50 pF.*

| Métrica | Cálculo | Resultado |
|---|---|---|
| Setup | T_clk ≥ 5.7 + 51.0 + 2.0 | **T_clk,min = 58.7 ns** |
| Frecuencia máxima real | Fc = 1 / 58.7 ns | **Fc ≈ 17.0 MHz** |
| Hold | 1.6 + 7.0 = 8.6 ns ≥ 0.5 ns | **Se cumple (slack 8.1 ns)** |

---

## 2. HDL

### Estructura del repositorio

```
src/
  top.sv          Instancia ambas FSM y las conecta con la Basys3
  mealy.sv        FSM Mealy: crédito y venta
  moore.sv        FSM Moore: mecanismo de entrega
  clck_psc.sv     Divisor de reloj (100 MHz → ~1.5 Hz)
sim/
  testbench.sv    Testbench que instancia top y verifica 14 casos
constraints/
  basys3.xdc      Pines de la Basys3
docs/
  P3_VICTOR.xlsx           Cálculos, hojas de datos y capturas de Vivado
  simulacion_esperada.png  Forma de onda de referencia
```

Jerarquía: `top` → `clck_psc` (my_clck), `mealy` (credito_venta), `moore` (entrega).

### Simulación

El testbench simula a una persona usando la máquina (monedas, compras, compra sin dinero, compra mientras está ocupada, entrega y tope de 3 $). **Las 14 pruebas pasan.** En Vivado, después de *Run Behavioral Simulation*, ejecutar `run all` en la Tcl Console.

### Resultados en Vivado (xc7a35tcpg236-1)

| Etapa | Resultado |
|---|---|
| RTL Analysis | Circuito genérico (RTL_REG, RTL_MUX, RTL_AND) con la jerarquía top → mealy / moore |
| Synthesis | Primitivas reales: FDCE, LUT2–LUT6, CARRY4, IBUF/OBUF, BUFG. La FSM Moore se re-codificó a **one-hot**; la ecuación de S'0 (7 compuertas en discreto) cabe en **una LUT6** |
| Implementation | **16 LUT (0.08 %)**, **36 FF (0.09 %)**, 22 IO · **WNS = 7.469 ns**, **WHS = 0.252 ns** → se cumplen los tiempos a 100 MHz |

### Uso en la Basys3

| Control | Señal | LED | Señal |
|---|---|---|---|
| btnU | M – moneda 1 $ | LD1–LD0 | Crédito (binario) |
| btnC | K – comprar | LD2 / LD3 / LD4 | V / Tp / E |
| sw0 | T – 0 simple (1 $), 1 especial (2 $) | LD8 / LD9 | M1 / M2 (motores) |
| sw1 | F – sensor de bandeja | LD10 / LD11 | Puerta / Busy |
| btnD | Reset | LD13–LD12 | Estado Moore (00 R, 01 SP, 10 ES, 11 EN) |
| | | LD15 | Reloj lento |

Mantener cada botón presionado hasta que LD15 cambie una vez.
