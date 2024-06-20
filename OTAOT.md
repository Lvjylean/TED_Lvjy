# TED module

## No.1 OTA5T

**电路图**
![电路图](https://github.com/Lvjylean/pictures/blob/main/module%20OTA5T.jpg)

```python
from pyted import *
from pyted.modules import PMOS, NMOS, PairedMosfet, ABBA, layout_mos
from modules.builtins import VSource, I, V, Capacitor
lmin = get_circuit().get_pdk().lmin()
# whether only INPUT AND OUTPUT can be connected between different module


@module
def OTA5T_1(VINN: Input,
            VINP: Input,
            OutN: Output,
            Vb:   Ctrl,):#
    # The first stage of the 2-stage amplifier
    M1 = NMOS(l=lmin, w=2e-6)
    M2 = NMOS(l=lmin, w=2e-6)
    M3 = PMOS(l=lmin, w=2e-6)
    M4 = PMOS(l=lmin, w=2e-6)
    M5 = NMOS(l=lmin, w=2e-6)
    Vcur, Vtail = Node(2)
    Vdd() >> M3 % Vcur >> Vcur
    Vdd() >> M4 % Vcur >> OutN
    Vcur >> M1 % VINP >> Vtail
    OutN >> M2 % VINN >> Vtail
    Vtail >> M5 % Vb >> Vss()


@module
def part_2(VIN: Input,
           VOUT: Output,
           Vbb: Ctrl,):
    # The 2nd stage of the 2-stage amplifier
    M6 = PMOS(l=lmin, w=2e-6)
    M7 = NMOS(l=lmin, w=2e-6)
    Vdd() >> M6 % VIN >> VOUT
    VOUT >> M7 % Vbb >> Vss()


@module
def two_stage_amp(VINN: Input,
                  VINP: Input,
                  Vbias: Ctrl,
                  Vout: Output):
    # The 2-stage amplifier definition with miller cap compensation
    first_stage = OTA5T_1()
    second_stage = part_2()
    (VINP, VINN) >> first_stage % Vbias >> "out1"
    "out1" >> second_stage % Vbias >> Vout
    ("out1", Vout) >> (Cc := Capacitor(c=1e-12))


def test_dc():
    @testbench
    def Sim_tran():
        from modules.stimuli import Vsin, Vdc
        from modules.builtins import VSource, I, V, ISource
        from pyted import watch, tran
        # The testbench for the 2-stage amplifier
        # The input is a 1.2V DC source
        (Vdd(), Vss()) >> VSource(dc=1.2)
        # The bias current source of the amplifier
        M8 = NMOS(l=lmin, w=2e-6)
        (Vdd(), "Vb") >> ISource(dc=10e-6)
        "Vb" >> M8 % "Vb" >> Vss()
        #The testbench of the amplifier
        ("VOUT", "VINP") >> (DUT := two_stage_amp()) % "Vb" >> ("VOUT")
        ("VOUT", Vss()) >> (C_load := Capacitor(c=1e-12))
        ("VINP") >> (VIn := Vsin(dc=0.6, amp=1, freq=1e5)) >> Vss()
        #The simulation
        debug("tran")
        watch("VOUT")
        watch("VINP")
        watch("Vb")
        raw = tran(stop=3e-5)
        debug(raw)
        debug(raw.get_signal_names())
        raw.show()

        dc(save_currents=True)
```
