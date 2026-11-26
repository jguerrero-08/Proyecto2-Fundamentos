# Proyecto2-Fundamentos
Codigo Inalambrico Proyecto
# Proyecto2-Fundamentos
Codigo Inalambrico Proyecto
# -*- coding: utf-8 -*-
# pyright: reportMissingImports=false, reportAttributeAccessIssue=false, reportUndefinedVariable=false
# (machine, network, time.sleep_ms... solo existen en MicroPython; el editor los marca
#  como error aunque en la Pico funcionan. En la compu esa parte nunca se ejecuta.)
"""
Proyecto II - Formula E CES Tec  (UN SOLO ARCHIVO)

  * En la computadora (Visual Studio Code, boton Run / python main.py):
      abre la interfaz grafica. Desde ella se instala el firmware en la Pico,
      se busca el carro en la red, se controla todo y se prueba el decrementador.
      Pruebas manuales por consola:  python main.py pruebas
  * En la Raspberry Pi Pico (MicroPython): este mismo archivo es el firmware.
      La GUI sube solo la parte de la Pico (desde el marcador de abajo).

Requisitos en la compu: Python 3 con tkinter. Para instalar el firmware se usa
mpremote (la GUI lo instala sola con pip si no esta).
"""
import sys

ES_PICO = sys.implementation.name == "micropython"

# ---------------------------------------------------------------
# PARTE DE LA COMPUTADORA
# ---------------------------------------------------------------
if not ES_PICO:
    import socket
    import threading
    import queue
    import subprocess
    import tempfile
    import json
    import re
    from concurrent.futures import ThreadPoolExecutor
    import os
    import time
    import tkinter as tk
    from tkinter import ttk, messagebox
    from datetime import datetime

    # ---------------- DATOS DE LA PESTAÑA ABOUT ----------------
    NOMBRE_PROGRAMA = "Formula E CES TEC - Control"
    VERSION = "2.0"
    INTEGRANTES = [
        # (institución, nombre completo, foto)
        ("Instituto Tecnológico de Costa Rica", "Jesus Mathias Guerrero Zamora", "foto1.png"),
        ("Instituto Tecnológico de Costa Rica", "Esteban González Montero", "foto2.png"),
    ]
    # Fotos incrustadas (PNG/GIF en base64). Si una foto no está aquí, se busca un
    # archivo con ese nombre junto a main.py. No pesan en la Pico: el firmware que se
    # instala solo incluye la parte de la Pico.
    FOTOS_B64 = {
        "foto1.png": (
            "iVBORw0KGgoAAAANSUhEUgAAAMgAAADICAMAAACahl6sAAABgFBMVEXf3tzf3Nvd3Nvd3Nrd3Nnb3Nne29nc29vc29nc29jb"
            "29jd2tjb2tnb2tjb2tfa2tfc2dja2dna2dfa2dbZ2dfb19XZ2NfZ2NbZ2NXZ19XY2NXY19bY19XY19PY1tXX19TX1tXX1tTW"
            "1tXX1tPY1NLW1dPW1dHW09HV1dHV1NTV1NLV1NDV09LU1NHU09LU09HU08/U0s/T09DT0tLT0tDS0tDT0s7V0M7S0dDS0c/S"
            "0c3SzsvR0M/R0M7R0MzRzMrQz83PzszOzcrOyMS/vLnZppCwqKPOnoqnnpiknJbNln/JkHfEk3rEjXO9j3mll4+gmJCblI3B"
            "iXG4h26yhW6phHSufWekfWqleGOlcGCTfXKVdWWZcV2NcWOTbVuPaFiJZ1d6aGGFYFJzXlZzVkxsTkNaVlZTTk1TS0hURT9K"
            "SUlHRUVFQkJCQUFDPz9APz8+Pj4/Ozo8Ozs7Ojo6OTk3Nzc2NTU0MzMxMDAvLi4tLS0sKywrKioqJycnJiclIyM7GteqAAAy"
            "z0lEQVR42t2di1sb17X2RelxaF1SVw4laRX3kEDckKZOWr5wiGODpbG4YwuBwBIXCYEkdEFCdyQu//p537X3jGZGIyCO0+f5"
            "zo5BAhTYv1n3tbdm+wJ6jFvDbx+j1hgxx6hjjNwy7D81X+/3HPpPm5MJTNjGFIb6PDXdG89cw/frggz/Z0H4G8YtGgcGP/n9"
            "fZPXALdzeJENAHGiTEw4QfTDtGv8nwUZdwz3H3Co0EBVGua4C8IBMvozlMvSMIGxdOz/a5CJqcEgTpQBHLca97A57lCr2w3e"
            "b2l4P8lU/xCWmf+TILeq1p1Odvi2McDYnTzW33To1iDVsqBcyuUbH8ThkIEnikx1+M6hX+wOQC7B9KYwAMQtHcGY+T8CYpOJ"
            "G2TcPwBFTXukpy93TN5Tv25FsSvXxETgDpDpO1TrA4F4GIp2v78qiFem5fZS9kt7z/lbGHwcHe/D8HJfPXPvn7/TfSmQ6Q8N"
            "MuR4OsL/zTc0PDrko3Xw//Xw5PcG0c7qPiBWMPHUqn53q/TdoT7yMKSf+pD3LoSMUCjo94k8vDAGgJjK1ZdteYHc5bX6QdyK"
            "M+ofn/CPKA2S+DCKr0YfiXZPLSz4ZkJz46Ojz0LGhE9ecC8b0SS3ua0PDQJ9GfFPTI2PjHD+mCbIRkf8fJiaMgxjKfSpb+g3"
            "Q8NTM4Z/iIz/MZCAt8saHMIxi9HAt1+Mj45/EsAXw3zpn/7x5L//HTSMP/xzOfzRn0Dnn/DPhYbGR5xB/haW8btB+Bk5sCuf"
            "f2+Q0fHxZ8FQyIAdGGHDmJtbGPf/4a9f//T11z+Ff1owvo6E/+3z/fnfr4IzoYnhwWnXe4L0V4t2kFuTrD7VGn0RogrFU5l0"
            "Kr4cDgaDC4G/guQP4fA/Q5F4PJOKGPhmMGQszAQC/gFRcUCBdbdy6flb4f29QJhv+OeCIImk0qlUJptJxSMzUCJKyHgbCqcy"
            "b43I8nIqHv7nRNB4Nj0yEcCv+N3v7g0iKO8H4gzr/Wm7XavgW4dnXoWMt5FUJh6mUDLZbCocgnUYoUgkngHYciTyFoyRr0PG"
            "1MxUYPzrbz5//Njv1cJw11kWya3Gbpp6n2rdH2TUHxgfmQtCGphoPB5PxZcw92wmEwkbQQNUeTzPpIkXj2TiRghKF1qY8gN/"
            "4lcEsUX1uysRBRKYCokUMtkclCoFBQq/jWezMJUwRJTJY2T5Abg4RWa8CoXm5ugXAqMT/arl1i9bEnyb+3WlW76JwP0aWlaK"
            "8nB0ziBFmiSYL+1j2XibyaVTGaErFCiSbJZyoaCyqdSSAflBeBHDPz7qKZEBrsujvpr2jCTTPw+EkW9igRypVDaXyxKklI7D"
            "Ry2nMXXOnqLICgEYQZrNF/KFTDhCF2YY8fBw4BeC2CXSi4fTPstp3dI8sRJZ5CWBmZkIPFVaJi2zLKRTtPqsiCGP7+QLpSJx"
            "CgX8XPAymSW44aARGn5r+MbviCXjgzNgjzTebez3BBmdGDfEKQkIRqFYwHypUPlsmuLICxsGeQrESqUATJuHlXwWWJ4Y/VVB"
            "BimWCyQwRL1Kp8mRU4okcy+IhVNA/KJYLhUBWCwVC8WiWH42HwmHF4Lhf/rCYZ8qnh2l1j1S4P5S12ns9wbhHx0fCafFoLP5"
            "XC5vo8CsOW9OHzDFSrkEhGKpVCwoRtgLXYIR8hmRXxnEf4upmxHEP2Rk4hlTDoWCTFwmT+EICedeLFcqFTzjN7KaU9xbZCn4"
            "kxF5MsGcHmFldOQWkIF1oqstfweIZtEOxqzXR18g+mVEUzi7YvGsVC7J7OVboknlcrlUrlYhE7BUSgJMDvxf8VQqDPf19Sdf"
            "fOIf9/nGfYMMZWAkcQcRcJgo9wHxj6oEa8q/jNQqqw2a8y/LtItlOjAgqG9AHlWSVPgPmFAzomTzjCupVORV0PfJF75gOGwM"
            "/xogni3GEZME0vDPhCQQ5k1tKZRk4rRrSkm4MHM8gKQG7QJIrYrnNJQi/pcivHYkw7zGgKmEhgzD5/dcALrFRtwgVtl+LxBU"
            "eEOj306FGLrhrMQ/0bvKKIhnKpZp3JVyUQsFAJRLDR/VgnJhwGEcheGUkKKFhvyj4Ynx8Q8OMqBxbckk8MUEruZyOpeVzIQz"
            "K5WFoiQeq0IOUTClXaJgSsmK5C3SGWcy+GBuU4g8G3/82Hj1u4D1B+8T2ftM3aNlejvI6AhqD8TBeDorDooc5TLFUqZQqFUl"
            "GAMCZEkwBIKPfCI/h85BaKV8JmIYYSQwxvTM78Ih3zjz3l8LxJ76mi021LOw1LTKE/O0CEw7k8cE6beoS3zk01oJGlaictGA"
            "qo16owg5VaokLBazqUgknIrEq/EXw75wkCDexh64RbVk1WpQ8+E2kNHhF+FQPIPMMJtRcRAXnp71rFblbMW+S/lymb6qRoZ8"
            "uVrH9/PFer3eKOSBJlzFguT9+Tiy/ghS4j/zT+lg8qFA+iK7FdP5hxZCYcQzFrUCQgPB9MrnWv/PKJc8VYkxBK4qW65Dl/J4"
            "qNXqReqbSIhhMc+oD8EuhyPxBV/Aiop9az6e69R2xXIuvd3Sv+4VVIF/QCCpTC6r0iuJ6TT2SjEPjyQCqVQLpWoxL2ZRqeSL"
            "NQgnX6rTbfHnyjPTcWUlL85ng77gUjw44h3eZT53+Kv3ABl9BkuH46XfzUp0o63z+lM4CChlelnhEr8LgyjiEZKp0/tSIGSB"
            "cyuU8pKEFfIZZMOIJvZM+JeBeJe69tWEkaFgOEVZ5KTqKxTPVOyrnlE4JDur8crzqsPQJXyUiyDCE1gIbacsbov5PdAhRTzN"
            "x0PT074IcmHv2D54FdF768N9QIJGSlWu2RwiQb54dlY5r5yVz8+YbeVzhbOz81qten4OU6mcYZQr9RoNnyANeDNyMM5TGkAo"
            "Kt3Kv/XNTRnx8eG7QLza8B4gt67yCMr4RAQ2Dt+L2hYqAYmcnWO65+f4XMjk5WsZ8lgkZ71+XiYHJFKnzZQJAgsqsHYsSoKZ"
            "iaT+5/FMMBIaHuCAB0hk0GaUe4D4h+LZNIopcKAOz1IM5+bM85kcQc56IGcwgUq1TnnUBKQKWxGDqoh10FCQz5Sy8dSy70Vw"
            "yRgOfFgQ/6DO7/DTBxEk75l8ji2RTI5505lMGgCFXP6MOiUc6oHzLFf5DPpWAwg+iXoxhtLXqZy5mE/Fp+eMJcMX8HtVJO6l"
            "nkELI+rpzEzPRrxTRvyFfwwZaYgip80Ek4etUINw8fFPq5OIQumbqWp1RkNomf6mBH+ViIm/Q5ryr1DYGPKPeyWNgVtclk0Q"
            "MwtzxLgPSGByLJwpYPaqSC9wwhJHRCbFXFaqxOLZaS5zelrUCJw7HxvNRp1OAdKClGjxCqTMtL60PGWEp2THwHuDvAi9uBtE"
            "UIbHR0OpHItZtg4Boi9wUcRwmt6H7cAHFcGRYUNbS6N4enpaUBLCs1wuB0SGfClWJD9jffIqbIwEoNV+7xTFO6g/6xv3AhmZ"
            "8hmYfi4r7UNM+kxZwxmmXjxPx1O5HCd6enx8nNzZ2d5JnZzKzE9OTjB9PpywMoRTOEewqZRV9ahIMm+lwT3ygUD0Suigsmr6"
            "LxG4zVyOfUP2Q4sCQk3CNygQZi6nx+nj/a3oxsZWbO8weXSUPjYHn6XT+Jw5hdEwY5HakXkyQsmfXwQfB0ZGxz1zrYFe671A"
            "Rv9qpOGpxNazOVEoKlYux+5WOp06hjplTnPHRzvRjbW1jY2N6NbW1g5okuA5StvGiZBQLDSVUjUbyceN4Nxf//rhQDyjiFkb"
            "fhvJSJWOMJI7M0fxVLp0x2lESvyXO05ub4CDY0NgdnYTiX2MJFiOUookcwo3VqvVtEzykfhSxAj9G4rV1/roT+EHh5CZmXuB"
            "sEUKUbAhVyiaGAKSQnWSztEgsscHmP3aiow1LZftHRkJCMaUTabYqCkQhPpsPmOEIsuh/wlMjDpWR34BiPe+TOV+F8Ipet4c"
            "G3A6cotEcqn9/ewpOM5OT5Lba6RYXFzUKCShikHJgHKg1QwF5lndBIFEMuFwPGL8NDFui+wDFt48gvrMjBfIoMLdj1o9lWcI"
            "gY9VHAUFcpJOxNPEyB3tbJBhUTggF6VdGATZ3t6mlikWpDmw9lqtUswUq9lUJpxKR4zg+Ih0HvsWQwekJp7yuBNkdASloY6E"
            "IhCJhTT246P9xH765CS5g7kvmmNlbWPNCQIUSkUZTPr4FJlLvVpIFUpcQc2nlozg8Kj/g4J4eS18BMOyxMZcUZKRosRCCCQJ"
            "kP3Etp742iI/rW1YFBaKIgEKXg7lwm+p1ypSjqUyea7Qj/h1deUEcSnWgFxx5r4ggelwVppyRYKoBB4cx+nkwQ5muri4vR3H"
            "tY7H91Op1H4ivrMtY2tje0sPfrljokjopwsus2GMbCscCn4gEL7QjWLLtCb84TxMXWWJRclOyJFMHiY2FtfWtjE1JCZcwZLF"
            "UYkuAMLcty0MxRGPC0k6d6Za3OUylGs5bMwM29fiPEAGllM093uD+IfCDCACor1W4fQ4sb27swVtSkAK+I+RW9IQOmOMk/R+"
            "PKFlo70wQHbiiTi1C9kaA2IeZWccqjU08mFBAu69QToejhhse+Qk31UCyaVh3zCGxe0EVIpKxQSEiZXmOM2BBInXjg0jkaBE"
            "4iTJsnlUzMThDeOGsTA06mpAOEE8lz5damWBEMVjmxN+/R/8/khWS0QqQXAktZvaPkzIFd+neoHlRDhyTHVPjlPJRIIAuxpj"
            "nyCCvZ+mkrIfE0+lloIBa8fNqCNB8awMb0G5C2QklMLFRlzXWVYxl9pQoW8NVxzmDhLxrYfIFQGjWE6Oj+DUdncteewnIbeU"
            "FmAW6okUNBWORMJ0vR8KJNAL7u4o4g/8JpxO52nuskCFGul4W0eNjZ3ttcW1DR0mEnsHB0BJHzNvR75Lb7C7B5TdXcTDBERG"
            "5TsGDIjYOX0bji8by+HAeB+HY/9GT7P6IWwoc3O3geCXjw9HUqmihHap1Iu5/TUd+za24bYQKpQtbDMVQcp7lEYykuQ4gET2"
            "ALK3J84qkzsVgcKtpVgSGE9AEjGe+T8cyABjV+b+ZDmSkrVOhkSoRGZ7UUkEDEhIEPJ2YlsSBpElMhM5EpDDw8MDyINjj143"
            "DQNigY/BJkY+m5p/6gunlp9NmP3+Ub8HyK19E6fzugNk6LmxlCoUBAQWf5pLb6woU0dKhXxkKxbbiq4tvmZuokmSIhCCkGRP"
            "NAu5SRYFvfjwAt15KvzZ2MOF8PT4yPCvAjLuSBgh9aFZw0gVC6qq4kiuaYkw0UWyHouR4/WiTtxh1UnFokD29oREjIf/e0bv"
            "KTJmngX+aMwNmTud7e+RsbV9B/R7HW0UqBXHbSABv//BCyOUkdYDo3Y2l0lYCeIKhbC1xbpQNE04RCRJLZLdGE0dye9+Uhw0"
            "rCNFV50FyNC3E8HQ8OgvBpnxBnF5LXzz/4XCsA42GbNc/EsnFu2ZroBIygX3JeHCAoFIYqhJYqoekRoxlWQKTMPPGJ9OLIQC"
            "Q7a98yaHI/ednrjtPXs256tBNIzHvl/UIxHZr8HuHOcSJwJUiSDQJkxVUl+AcMb7ollHYvCHu/QBW1IjqvIeEAmVBKdCL4Nz"
            "49YubndhZYFMTdzVPNHyuBNkdHgaJXtalng4FwF5LSAikCgEwsR927QORcFxuLVC2K2dA82RFA6JiRH/H/0jw3eDTLwfSMC1"
            "01QiydBbbjPjorRoeA9kTfolEg/hXXW2CDs4ki7Q0dHexsrr1ytrmzuHuiOkSAgSD/uGf+N4W4YbxL2bybbFoRdEfh6Iz0AN"
            "kVYgLDx6EiHI7iGKxOPM6am0F09VWBcQCAQgiyvrW0kV7bVqxePb8bjhs7294QOD6HTL77b44elUHEWEEsj+/rYCUSLZIIik"
            "itIpPUvF95FtCcrBFsINQVY3D05Iom2dINuRed+QG8T5tgunWjm2k/Z73jtBHj2CTP7oW+JiMtRr3wWiBkorag5gzk7gWuXq"
            "Hx0dbLAhISDrMfkeEzBeCnBsLz30+X4pCGTiDeLZE/rjRw+GfQ+NcAQuS1zn/vaKcPTG4to2At4+YPRsj2AkR1sSXPBSgGwm"
            "kJ+olaDTdCoRf7Md9jlBbMs8faY+KKSbejXjIRF36f7JJ/7hB5OhUGhyEgLZjrOq2FjsB5EAIt2eY5UyHh0gf6HU8POV9fWN"
            "dC533uRSQ/Esl46/idhBnFHEJZHpWzBMUbyQzzO3gviHZjmbxcVwPLu/sc3SqA8E2QkyXKbwIgqCHB1urliCg0jWd06yp+fN"
            "xnkmlTktZiIfFORFH4hlJybHx2MPQyuxd+9ii6/XNvZ3ELtRiHuALG5EY8ipDjkkkhxGV3ogtJINOGBkvzn44OzZWVpATHPv"
            "bTF37nnoNU+cBHO9YG6NFxi3gIx/FNp4J2PlNTu6i4s7+9tr/Rzage2w9jhAebUXXVy04cIDryfEJZ+opZRceum3vzaIw9g/"
            "GVt8pwdj9Ao91HafQHphPrYVY9oei7r8wcrKepRVsERFKRHfDPVAbvVa/SmWo+Ewcy+Qx8OhTULEVlZjsRXpUW8ntlc8SCQT"
            "XttkoEdWv7rqAoFybezQExwLBvxv+FcHsaVbjx+PLcJA3m3ikm6+WxeQLd2u7iMByPpmNBrd3FxfX11xe2iIZCOh8rAkQd4s"
            "jTlVy7as4Cxyb4Gw+ywbiH6nXw/k8af/9XwdHFFMcz32Lrqyiki9QVv3BkG4wIA0VrShL9pBVje2EyoOMbK/efPcN/T+IHNz"
            "t4A4U2DAPB77jRGFXq0yEkRjq5jpCvsNuq5yc6ysgmRdc7hAiIn8OCGDjYo3b0K9XKtfsQaC2GXiMHQBsb3r0g7yyacPX8fe"
            "xTaZwa5ubr7GPNeQk1srCH0g6+smB3WL2cnrRYWzuCgg0jXaUSAG5DFkGskAEPUevQGl1Ny9QR5/PLnyLhZdJwgnBs0hiDQZ"
            "V5wgsky1ymE9WPWw6Qv0UlxCekdv3oQf+d4fhKp1F0gv5Rp7+HwVirWqNCO6F9vcXFmMsvGjHJjDZVEKq2qsi7XroV+2uLpq"
            "rimKSDbevJmHbg0Ne4AEJnqq5eGxbJPvMdwK8tkInG+MJcXqZuzd0dG7KOx8owdiFbx63dDGAZGYLK8tkK1eQ5skxq8OYrIE"
            "fh/ajG1Cx1fAsXeItGONXdItWYO2dMe69ismx6aQ9JyXiGx9zQ0SfuDzAvE72vC2dU+3433RNwaCfDy7FlvFZQfHuwNkUcw7"
            "1ral+7O2tuIeAkF50Alr7dLms7a+sbm5Fdtxksza/dYHApmybtRhS4I/HZtcRO63uBLVICuSsavuj5vE1KpN4dgUPywBBNFe"
            "rVLHZP47uk8MkJCv3/06NgW5SxF3JB8MYsZEdVOkv4x9/HoT02H6+w669Y56YoLoPQ422yCGptiMEkRqKkrz3W4sFmOfToMo"
            "EhqJfsuv8xYpvdVcN8StGAsLA0E++0OIMXBVQBBQRNm3t/Ry7bpCWdMI6yYHshRmKqJa6+BguxEcwCGCuF9F8uFBnk33aRZV"
            "68+fPwotWiDR1+vio0wQgVh3DiUNSbgQflYhjsOT09OTo92dd6hWgJDY3k4oyQjISO+eC469/RM2EKeNeDOYqjUI5NOxWbjN"
            "RVEtWP3mKr3Ulk0ievImhQWCz7FNatXxabt9jjqeNdfBTuI4ubO9q2P79nboXiBuidwGMq3vWKXuj2Yz98/GoOVKJLFoNBZj"
            "+brBtXWayfr6Gj3tpo3ENBCCRJGmHUkzAvVv8iCZPDg4OT3a29lNaIkgIj6w3QVjtE+xpgapVr9ScdwK8jC0KCRRygR5F1L5"
            "DYiEXdKNtfU18VLrbpBNZo8xugd2TbmwmMSnwySZ9nYSu9pG3kz6hm8DkVshvA+IwDjiyPjYo1mdfYjzWVvZ3JBur5CIYq05"
            "7UNxMH+MMhfQIGqFga2JZEKt8RJk6YEdxH5DlLskMki/PEE+4xgbe2QoEKSMqBEX1wVERLK27jR2O4f45M13R8fsyEtD4gB1"
            "CEAOdnoghi2MDACZmvp5IKqFJ6K0YvyYHkok0mdbh7GrXTIwkvUeyJqDpZcvrm3ssBPBj32AQNEOzHV3m9NyKpZjn4BnmtWv"
            "VdYTAXlm3qVO/5Ynakz+yWC/8PUinNHW5lp0N8oOg5i6GUfW7CD2xJeDCRZw9lHgHib3EztWhkJb7zUZvRdz+9ITb3ksmMMn"
            "r/YGmZgcC7MkVEbybm8XwthRIGt9IOvrTo41GFJU1kdhJ5rDCu1vxobvBJl6DxBTtVwgTybHJkGCSUV33+292+LkBGTDDiIo"
            "8FWrPRDZo8k0C9OmcnE9LmFuTaFmhYd+21tPcN79qCeRZwNsxMtKPEGmpkyQJ589+ZNBmWBa0ZXXykiiThCkKetmwqUZ1iTW"
            "RKGGO7tqvwCEsp/Q2ZaA+H73nwSBA/v8s49Cb1SblAu33KEszld0atU9RKGisi1T7c3c4caHPbXWa5IoW3/ojiLjbtVSCM/6"
            "s3cHyG2qZatQ/va3wOQjqBekAv1CUNQSMY3DBULBbcVk8GXElpV2tcC4Lz2UHZWgeMX1DwTiLkzMoDLx2Ufz4e0tpLAHhztw"
            "RFsbWiR9MkH5pBL23ZiZlG3JN5AtSluLJGLrs76PPFXLo8q1cbzwznvtXuuZGU2m3TDQ2idf/GkkFH6DGiKc0CDK/666QKLS"
            "+wVIjLq1FWPWSybZCUVLOdxPxCWujw0NW3f1sNWGXr3rW7ITb5BnA0ECn08++Wjo95/+1heSjYqq2l3vA4FAdmX/ye4OJCPJ"
            "FtOUvV1xwCSheimn9eCBy/l+OBB1SyF1+xRnDQ/nFXgy8RfkLJPcP7plplpwvHaOdW5zAMmB+Fu9Rn10bK277ytDiYvTevgb"
            "zxU3pVzT9m1Nt+QmAvDqlVsiA0F0PoyUZeiN7ITF2Fzf6IXBHoiSg1pO1AvVGgPPpPebTCin9WuDOPTLqWGfjT0ZCu/INljW"
            "gSovURzysQkT2TuQRXVuHeDargJQ4+QknZQdA9r7unrXfe2TaVeX1DPTssarVz77/UDvAJkYNphlbAvIpq5HTKEogYhOsfaA"
            "acBzMbeRDPjw4Ih7BpJqh3PI9/DBhwJREgHIzMwLeaGLRd5o7dKvzz+Z55Z9TWIm75pkXXwWjH1L3gsjwSS6yWUspml7iX2S"
            "KGN/M/ng4UNnAHHtNZvqb2jpdc8+gbzSw2e+8B4ggb89fsNOiOyxVtW5gKhPBMGISmgXjl1z5Q4g3DjEdZ6EVFWPHjx0lerj"
            "gfcEMVH6QEwcj9AYmPivpZ1tprFbGiVqVVSs6iWiq44cIZT/OtTjQEKJmEjY1w8S8Lwx262q9coukB6Ik8YyFQfLl783VJfN"
            "3MCvYaLRXmoiC4ng2Ds4POqN5OGB2qvFiGj4Pn740FseE32bru3Zooelvx/I1B8N9qaIoqasSKKmOPTY3aUuHVpeS4EcqCWr"
            "OKqqYb/H+0V+KcjcnBeIHq7c68s/hmQJTUpW2DH/SbKrpo+hHpwcJomIJB7fXvrs944A4rX7xN2Z8wjrbver1n/uB/Lkk/k3"
            "LF4TiV2VUlGXLDHoIbYtb0bUKEq1TJCd7fDwn34hiJTq/SB6aBAHjSv3+vLz75e2ULwe7Mm65o5DnXYtD8U0RVpaloXIhuYD"
            "tRMwEvqtf9Tz/d62/aTuKpfZe78cXKplQ3FLxB0bv5wKb22jCsesZEfvjl0ce9Y4oGr1SGTLqQWy9PnHtq073vsXFYilKHaQ"
            "V68cND0Qm8T67V7f+tQCefJFeAO6JVvf1U7r3ninGBQin4qlJFVzSzyx2gloDP/FHtAHvs9Qr3u6rMNTGq9eBYN2EGsNXrP8"
            "69k//vGt+tVfffV30rCtApID2TaupWJSvDPl8U6MntazJVkLhbGnQOB9I7MPH3v0sWxvpLK7qxd91tGb/0JQjwU+82nTsYPM"
            "WCBPn/7jqYyvvvpqkmPM4D6gxKHSGEHZE/PA/PUQiqgZJlG3q7I9oTc+hIfGHn0IkGBvmCBaXjYWS7ee2oYCCW0ISfJIt0QV"
            "y54A7GoPvMU1KxZfAkKShH7TGwTy/NHYmO0u8c5urw1kQOMa0+wDkfE+ICDZ49t3jo96LO92tUKJONb1mqJEfNkgr0ji8fAD"
            "1DW2W5j8UpBXDhBJhV2mApKnrkGS59x4DdU/SJpBW6K2MhaVa0mKL7mkJC676j096g18kaeP6W4dR3T0mr3THns1vOKgBWJD"
            "+rkgsxvSRASJVTSxMAeKsnErLVb5l+mWNUjcePDZvUAslzMQpOe3NIhVnvS7r1k1bCizAFlfk/er75HE3HZ9JG8XUXa+FQMA"
            "068tlXSJwyIJBLI0Fvisty834F1M6b0mfXsbPDMTN0jQC2SuH2R2dnF9TfpwIEkeWaW5RtF2ssslLnZSZI/jYVKCYXwnEXn+"
            "5dSTwIQDxLyhjgaZsdLEAWuFA0Esr7zgQuGDgHyPIRD49P3k4uaa7Jbb2rVINIuJQjkcSDElxUhSUhSJ6safn/GNFBP9h6V4"
            "NoDUTDzaDdaU3SAWoXILSqZKJN9/r1G+5wN0K2yCWK0GC0ahmGkKKY5szaDEfvjP6mgBO4a5eGnf0DQ357Gw42SABr0KOtyw"
            "J4ilX7Oz+IUkwC/+4fvvviOIdOa8SGQNV2cjB+aOZn07Dm5mjHwzaV+k/PkgjpTkldv92n7aCydqmHol4pj94QdK5PPXqhO/"
            "KS4J2bqD5ORYvwvRLEVOpMd1fMK30Mw/nlJv0vG6KdD0M5divXCCWIrUFwkHg1gwbhBYiQKRGLElJIwnThRbJ8v6yXE6vW98"
            "/Pf/fn+QhRf3AdGRxa5hlqlrM1Egs7OTi2rTg9KtXamgnCS2wT7dqTwcp9+MfffFE4+7/PW/D8y7uHVGQg+QoB3k5cuXeDUf"
            "Xv6IfxgCMjcHGBvIqhJJTEhQ1B7bUU41g32cpPeXPv/XlxPOHoAYxwcDmZ/nw/z8PF/0HONlkBwvn+MfBuZOnh9+UByz3ysQ"
            "FbqFRMz62C6GvnFyvJ9Y+vS7L7/UeysG3yNE2bpG6I8lmFfwThCS4PPz5/PPZVggFA0RBOTzxdU1KyGMqqB36FAvNwQ9wD6C"
            "+qc/fPWVrQMw+L3EjOyu+GGGweBAgRCkbwwGefo0vNHbZqZE8k6hHB+7WUw7Z38xkQjbQKzzDe0guii1tsN6Nd7tMw+p4Q3y"
            "3Byzz18KiEhBP8w+n5xPnaxb2xfZJY1Z+ZTLUiwXllQ9eIB89xXHlz2QAe/1lDLCiUGdernAGbo5zDE/CGTWfLSNSSN9cbhm"
            "2/oX7ZGwA3R0bBcLKSCNw33Vmgv/7Yd+EC+MFyIRD5CgmmOQ/xQHz6Qwh89bryyWHsb8ZCjdOo2u9zYxbpokqhW0t9dbDekl"
            "xfp9rQT55ptvvhL10vssnJXHszkrbe9vyCkjl0s/H5o3KaxxfxA8eRRKnV8cbbpAovbenF6vOu5vlO4Q5Ntvv/nq719NsY+B"
            "p0oK3839+OIl/P3Cyx+lAvEGeSnmYVclwz18g2z9uU23ns+ixl1K5Vrnu9HVfpAeCcstW+daFfRSjADkx+++++abf33792++"
            "+fvf//Xd9z/8OPtidu77bz7/9OOPHz548HBsembhxYwu6lwkL6lSHhjh8M8AgeFPzhpLqUwm1zzhG3b0qqENxC0UqyvXq6oS"
            "4e9eIlf77nvEpB9f/jj71eTYw4eY/SwmFeahK8thY2F45NmrBSfIS2Uk8/N9YgjbhyeIDWX+OfVrUu5OnsmeNY9j0VW7SOxt"
            "eLPBdZC0OKBXBFGq9RC/dvbp5KMHQ0MPxp7Oh4xwJJ5KZwrn9dbFRefionWeTS0FRwLIkPqC+qt5F0a4bwgITcgDRIXH2dlI"
            "tiB3ODg7P9yK6rVPKyg6QUSVDo+SurloCgQkKS6ujz3X88/myw1Mv9O5aLes0b68KKTC306Z1epPP/30QkUBk6Nv+ktLXiAW"
            "Tw9k/ulkKA4OuWHx2fmegCgSh2pp16WLKrkHh9lB1SCnl3GeenPWbF9eXl2SQMbFhfyT52C56LZSxifBBVtO8lJNLeTmWLIP"
            "DeLWrN6nWWNZ3XiZt304PdvTqrVugdhXd8xOgxSHJBHF2tMgN5f1qyvqkEwfwuh2KJELflx08QM+abfa3VpkYm7BpFDho49j"
            "qW+4Qeik50MK4unT+QiNoyBH7uRzufNDZezrq+b2vx6JAlFLhgcKwCYQgly1Cu3LbvfyEp9kXKqv1OfrazwBzQU0LLXwzSsr"
            "eCARtOnV0qDhM7UqpJWLWdXT72GXofDysnDwLsO80XAue34Ui9oo7KpldrAO1H5fBXKgHhO7uwJy08o3r66v8e/68lrm7hiX"
            "AkWruc6E/qVycB09evZxD5AQZQBvCF8YgTtbNg8UyuSLBXWz+GzhZDdqysNswpmLVtaSW1Lv+FUcezaQ65tatlBv4/pf3/QG"
            "n5NMgVxfduHALjOh74IKxMlxG4hOuhAxQnQoct5RNqNuYSYC4eEbcipErnC6F7U2lYpmxWK2mEjdMpd3xPuK32IU2d1LnF5B"
            "JNeVArxGqVKptVpdoRGNuug22w1YjpISvoZMfnBg3CkQBYJoAbcoR1OpuasjNoQJDCU554U3tj07im72OOwruubaoe394EnV"
            "FAYIe7+5NknaxSq8RrFWr/FextV6k3bfbjaa5Vap2Wy3Op0bCqXTukz9Jfgq2Ised2AA5Onk9/M8es6kUAcHqdseZTOFUgl/"
            "t8EbQOeL+HccU5s2TMWyVtgtp6VBdJtLgZDk+PwSIJelaqNahJJWmq1Wk3dmLlXgdFutWqPSKNYr9Wbzmvp10eos/62XjdyJ"
            "ARCYRFxuiKeP2MqqW4hn5YigYrXCW4rz3u+FYqVcyEO3NvV7LKLWiPWSeXpeWcuVjTUHPZC99GnrCqNSrjVbzTKsrlSuNFoQ"
            "RoU3PoZI2tlmtpGvVFoXVLd2p2AE7Rh3cCz5suoIMHWf6Lw8E3fLg5141A6ZYCUEOT/L509ievdJ1A1iulstkmPVozc59o5O"
            "z6FbV81ao9kkQJU3YS5U2p12CyzZQilfz1ezlUwlq+wETvgHRzpyF4g66ExkUChXxBLUOSn4K5AB/1pRzqmoVCu1SiGfjopA"
            "og7FspzvgW1VWie/YiN7B0cn5+0bxJJmu9ZqNJotCerNYqGMaI7gwSM/ypxEppAV79xtt8KOKDgAYHl5WYHwCD018aI6SYRC"
            "oHLxzqmFAnS4Vi6Wq6VStVqu1mpnxTStxCaQLWeiZecwQRKJvf2jk9MWQWqXwGjAQFpN8jTL5VodQmo1qrw1foZnL2QadGYX"
            "16mQPZrfBZLteSp19g4Mu1TMyxlO0K9SpVrjORY8G6ECS22en6c3+zSrtzxtBpIjq6OtNOvg6BhGcnNzUb28gHZBHBcgqVap"
            "aHKQD7QYDqWQwd/MdBkfuxfN8B3yWJahVUsOC8qbdqIOa+NBSJVqnccHVKpyKIKcKlAqlirNZuXseCtqH1uWZmkrMW9LZRPI"
            "AVTrtEmQSrcLB9Vm3t6CdeDaNNRZPo1GuQiNYEqUub6+oUjid9i5A0T7WTmBTh9vVuL1KcvxFOV6Hb+eZwmVCmWeOlJtNs7O"
            "T3a3tpy6ZYJwUf3wwMy4dDJMgRykAXJ5c9MutS9bpWq7eyEJfKPWaDRqEBGe16ALlQbdZLmGYHndvc47MTBnc9qWVi3rseTT"
            "EUPCB8yAR+3w1tsF3t0dALVGVZ2MxDvxl3juznmtUMztcRHaaSNasbhAZWWOvcWS5PHJ6fkFQMqoOqpV1FJMg1mFgIGGf9Hk"
            "bdib7Tr+bKbAmI8IH3FIgxTL+kERLS/3QHiciBzoJO5XTmqqQOC1Ro03RM9XK4TTh1YVccGoa+V64WjH2snf49DeN2nnSKgf"
            "HBEEbqtVbV93WxWQIGOXMoRMnQ60jDcwb7ThAcrI70jSuUnZxGHXJqVRy/bhk2NaCsVsVh3IyHNS8Cvr1WqJR2wUStV6RZ1V"
            "BQ7qXJU3Gq+fnR3vWtvl1N4gBaJX2nokCb5ZQRYfAHJ106i2rq8vmtV6S8qqLusqGj4uUB1+pUYhIfBm29Stm2y43yQGDF+h"
            "oI9jk0c58qVKFeONUwuMvrWyHMPDEzd4hlul3mjUG/Xz89Oks1pXSXzSqtgPraqK1s9biTUvrq4a1eYlRFJtCAhLK4knPIgM"
            "v7hFx9xuMLer3dxcXjUjDoO+DcWnD6HgIWYlOfQERl9u1muVspwVxqNQqnAmPOICWR4CMjkarUa9eXa8v7trozADonVbQCkT"
            "Eyq2HCuQy3q9eX1z2apJSARJ97ID7VIeuFGr0+jb7Sr+XrbQvb68iveseRDBkqVapXK1IYcB1Wo8wqGYlfAh5zkUi/BS8MOi"
            "f0i8SnWocgOeEjaKNKlIFDtHDyMp79ujPPb25M7+p6dnze5VC8EQYbvd0CJhkQsxQFkbcpoaSKBoDTicUj5bv4SRDLKJPonQ"
            "viu1ijoHDBemWDazFGQmxRLyCTkHqazOEivXYemwxyYvXbPeLB7v2TediWpZ8oBAEhLsj+W2s+etyy79bJf5h8pSaPCMJ80a"
            "Lg4PJqJmwfLLcnpUtnCTv332DhuRg4+oRbAFRieaA889gIpB23ipeON9xl1GmDIdP0ladJ4AOj0+3DNdlrwT1A5yoJIWE6SL"
            "LAteqsuCo6G0iPYuCX0Ff6lOkBZiDBSYh/xkC+dv7wPBF/mKclapnEpVKgLEPORQnG+hAjdMvePRTmSl3DB7mIl4TfGf5ydH"
            "B3q5x76HUbamHch6gyyZnDdJDnu+QHrbbQuH8lgtJsBQcB4XBa3tALNMf1OvZvORpTf3AHlLiRTFZcmhWRVWbjxdRx3aAhT8"
            "9pKc5MKcUc4NVAUFdUv1pC6gChALgkfS1vVVNwpimqIXs07PzyX24V8HlRN0q6k42oDDL6yqA1YwgeZF57KLiFJDPVzOp2SW"
            "90DxqaPB5OwcObONDhbRFV6Lt7Tl4YVAggVVG1Bbnk6HfwgCrOrgdKRh0OlctnhrQ9t6Qg/oSN2VmQJRWUm72+myBFQSAQfx"
            "mg2ef8G7/Ndrrc410rFms15vwRHHoThv74QBSEkdnqfO0GOapU8+KKPy0SBwJrUavBXrUkKVylCQNnsE0Peu6uLgSQuxxVpw"
            "s4gUBzHworbqjFIkolL8hqQoEAlPV0IVX6s2O/hdAKnWWu1qLfPWZQyDQKo8z6wgNsDYXWN8JQsPO6ryLNMSMjl4LaZgDDM8"
            "Vg+OmlbZkSgA22UNi08skOpCYwGJTsGlXuLHqF9bSiSqN9duazDVLYUEqjxqrU4jaeMP1FlIlmrxpbsFIiASxmEf/CU8eKZS"
            "ldDdrOGaMLrLMZo0IJVwy3GAFabdSIzAcIkPVA9kuZHP0JqmeRgU1KYrP1E/kOQKIJfSKSWDNK9pa13m9bUqw0kT0u00KpRV"
            "u1tpZt+Kmbwd5K7eamNnVsjDs+Qsxqpw1OlixaDrtAieR1co00gYgGGGTTpgvoZ/SWHgguPBarpdKbZL/vBGvo+vuvCrkudS"
            "E4WIU1WWQgPvQgbNOkUCgbVr4hku8Dfig0AEQQ2CyKk5UCexc549R9XiLJEBN3icYU2fqcMojySFAz+tVVFN0Hd1rwSEndAb"
            "91C9UHlyRa+ACQO9SxPpKJdFHsJ1SNeSS9SgJl6okAJdvcgQ5K3LRJQc3qoPDh8sQSZa4eQRJHDZMXXkQg0pQ6XugVhQvcuJ"
            "R/wePrGnBvWTyHhxqXrQvPb9LAoGGsipikhoU/iyw6867Ft35AsMRVJvMGSy1cIEptpqipXoK2/Zy1vX8DWkZqadyQGAtLk6"
            "3HmdwUsCX6MuNiExkoCKDordklRFkhXkFZjdjde4EjOHLCiSC75QvnHVUZkvfkQW1Z0HGX89rAQvbdIZdDtw9TnnlJVN9IGg"
            "xKxXSxUeOaMqTlo6TwSDYKQFxXlTRPAGFEqLACoI4KPJaCbtHVjo1XWfMG547fGfctKXfIDZ03y6UiJeUEqwGsiEP6dPpqQp"
            "ElwrSg/OsZFaenvn8GEerJZoDA0lhSqdRxNSoo00pXXDnLdeRjCp1ZAwqZ4UA0KXK0+iX8pqO11tKaJkdGKYYJdLVHrNoNO5"
            "UhLhSoikjPBX4vu6aomk05JrAzR6MxFip+ESydtlT5A61VJO/eO02Z1B2KjQdcGylUxaSOVhEE1JXVpSo15gOhcqtHeEX1yp"
            "EF3oyGIO7dMw+y5RRSy0DFyODl7K55caA6hwZ7x2FyCiC0A12W0248t3CSTia7E0aNHEmi1JosAl5sKj88TLNpu6S9CoSrXL"
            "xA5B7QIzpJ7zesIGWm2zl9DGD+l3lQpdXWkZ0C11VOyUVSplL+AjiF6yUi9qK/FKWnp92bpstzPLd6tWoykmyzSnoSKEOCv6"
            "c3xC8kuZSMejKQ21erUujrHJFRtOVKk9Zs2YJgGizavd6XRMe7hSKYzM84odK+bxHYmMsu5Gh9ZbtmJkpHyRH+P/ur5uQQfr"
            "kYGSsBm7TJ+nSZogtHY5b0oFvZaWSEte2iIJ+511OLgG8j+lERSMaeuUxbW5HHUts+1on3spTqyrlkA72v6hYNc31vKbZGGU"
            "b6MJT3Z9DQG3LlPLkbskwv+pocozGrdMssk0Uc6WBIiC6KjKAXNnp5ZOTSJKk3pOrejqFc3r3nqauR6FmQsmLberPJe5GKrH"
            "lYpBfPGl/CqRa7OJp9c3F+3L+lXmbeRur9VWloGpI4Y0W5UqE8+GnACG+rwq1tHqdqWKgrOuQnp4JUHYK6DL6lAw11r7RZlU"
            "xsJoryYtJi+SaUtg7KrIQaWicl05lkYhIunYNbqMsl2AXDYid9oIU1BaNaZWk/448xVRLslWaNtt7VylEAI1Unk4OMaapoR6"
            "cZjtDvNHatC1TExnW1fKjm+ulTu4kACufK5alqao+arrXkpzSfvqomwUkMs27P0q5RZJxPXExzqTF7emPDCfSdLYoILBAyBY"
            "QjhN5Y8uZBrSRq/ToUlsqYq6qYScc5RJih6p50wBAd5SCQ29s6VYDJYdxkMzJxMWpXpIhtvku7y4bHVuoFveQuG3+eGrtqS4"
            "YTVQ4eqk8le0e5Wwl1X+K86Z4Zv+ptuGLVXZfK7SvBh1WHvhaZtxQGlGqyGRtiKJv2pZiEtvtnsgcLEqiniAdC8abfk+QC6u"
            "zyIRDxLb93w11UdmIJSao9mi06VeNZizo3ZW+TvdFPxXV3li3UlpCiCkWFWnyzP7r1dU4ixFAQTGRmhT0lx2TmDCHdE3rWN0"
            "y1fiJ24skGtt8+0WCW8A0r6+iLtEIoIw6SLK2BFKERrqcpivXPkGU/daU7SNjrmsUhhmkowPXcyGiQlyopb02Vq8CNJOlawf"
            "+Se7SMw4Jdy3mdx2GNZV2qFMR4MolstrW+asnSB8l1rxuW5fXN+kll0g5iAgfvK/d0WYXtaxA3wAAAAASUVORK5CYII="
        ),
        "foto2.png": (
            "iVBORw0KGgoAAAANSUhEUgAAAMgAAADICAMAAACahl6sAAABgFBMVEXEloatjIO6gmSbgXu3eV2teVumd1qDeIGvcVejclWr"
            "a1OialCablOZaE6HbGFwa3yiZE2lXU2ZZEuYXkiRY0uRX0iQW0aPWEOHYUmFXEWIWEKBWEFqYm95WENhXmtcWGaMVEGOTT6D"
            "Uz+DTTt8Uz58UDx8Tjp7STdzUz1xTzp0TDdvSzdySDRtRzRvQzFmSzdmRjNoQzBiQjBWU2BQTVhVRT5GRE5kPy1iOytdPi1d"
            "OilVPS1VOilJPDk9PERaNidTNSVONSdOMCJJNSdHMCM+Mi0zMjhEKx89Kh83KB4tKSw2IhouIhouHhcsGhMjISQgHBshGRUa"
            "GRsfFhEZFhUWFRYVFBUaEg8VERATEhMTEA8RERIREA4QDw8UDQwRDQsQDQ0PDQ0ODQ0ODAwMDA4NDAoNCwwLCwwMCwkPCgkM"
            "CgsLCQoKCgsKCQgNBwcKCAkJCgsJCQkICQsICAoICAgHCAoICAYJBwgHBwkHBwcGBwkIBgcGBggHBgYFBQYFBAV2ZncPAABX"
            "jklEQVR42n19B0Mbyba03r611xjz7CUHg72YnNOaeAEhDIgsJJRzlkYBGVnBQtJf/6pOjwTee+/XtokydE2dUOd0T4/B4bA5"
            "nLlUKhmIpYKBQDJZigRTyWap1syHvF53yO13+x1+v/Ua4+Li4uzs9PjYZNozGnf3djFMx8dnZ2YMfO/i6qr19vT09Ni4t7O6"
            "ujgzMjgyhDEyMjTIdxNTkyMTEyMYE7Nz86syNmVsHRzs7GxubeEjfLyzc3B4dHSEH3R6dnZ2cXV9bbFarZiJ35/KpQK5VC6b"
            "TQUCKXzqT6UCfgO+6XC4A6lcIInPYwCSDEayzWatXk6GACnF/5lyXD8DESSmY6PRZDKazswtAFfyXgARyJFxb3dnZW52YmQQ"
            "AIACA1AwfwWD7+fmFwGD/za3dra2dnYEBWBs8vOdw8Ojw6PTIyLBz768utaB5HDdU7lcNhbEtPkFf4BAbDarxeEPpAKpZCwS"
            "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"
        ),
    }
    GUIA = (
        "1. Conecta la Pico por USB y pulsa «Instalar firmware en la Pico». Solo se hace una vez\n"
        "    (cierra Thonny antes). Después la Pico arranca sola con la batería.\n"
        "2. Pulsa «Buscar carro»: la IP se llena sola (o escríbela a mano). Puerto 8001.\n"
        "3. Conectar y controlar: avance, dirección, luces, sensor LDR y patrones.\n"
        "4. Pestaña Decrementador: actívalo para que la fotorresistencia (8 rangos, 0-7)\n"
        "    alimente el circuito, o prueba combinaciones ABC con el teclado.\n"
        "    LED: delanteras = X (MSB), direccionales = Y, traseras = Z (LSB).\n"
        "5. DETENER VEHÍCULO apaga los motores en cualquier momento.\n"
        "6. Para cambiar de red WiFi usa la pestaña WiFi (con la Pico por USB).\n"
        "Pruebas por consola:  python main.py pruebas"
    )

    PUERTO_TCP = 8001
    PUERTO_BUSQUEDA = 8002
    # El marcador se arma por partes para que solo aparezca completo en el comentario real.
    MARCA_INI = "# ==== INICIO " + "FIRMWARE ===="


    # ---------------------------------------------------------------
    # UTILIDADES DE RED / INSTALACIÓN
    # ---------------------------------------------------------------
    def ips_locales(destino=None):
        """IPs IPv4 de esta compu (sin 127.x ni 169.254.x)."""
        ips = []
        try:
            ips += socket.gethostbyname_ex(socket.gethostname())[2]
        except OSError:
            pass
        for objetivo in ([destino] if destino else []) + ["192.168.1.1", "8.8.8.8"]:
            try:
                s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
                s.connect((objetivo, 9))          # no envía nada: solo elige la interfaz
                ips.append(s.getsockname()[0])
                s.close()
            except OSError:
                pass
        unicas = []
        for ip in ips:
            if ip not in unicas and not ip.startswith(("127.", "169.254.", "0.")):
                unicas.append(ip)
        return unicas


    def _udp_busqueda(local_ip, timeout):
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        s.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
        s.settimeout(timeout)
        try:
            s.bind((local_ip, 0))                  # sale por esa interfaz
            base = local_ip.rsplit(".", 1)[0]
            for destino in ("255.255.255.255", base + ".255"):
                try:
                    s.sendto(b"FORMULAE?", (destino, PUERTO_BUSQUEDA))
                except OSError:
                    pass
            datos, addr = s.recvfrom(64)
            if datos.startswith(b"FORMULAE"):
                return addr[0]
        except OSError:
            pass
        finally:
            s.close()
        return None


    def _puerto_abierto(ip, puerto, timeout):
        try:
            with socket.create_connection((ip, puerto), timeout=timeout):
                return True
        except OSError:
            return False


    def escanear_subred(local_ip, timeout=0.5):
        """Prueba el puerto de la Pico en toda la subred /24 (no depende del firewall UDP)."""
        base = local_ip.rsplit(".", 1)[0]
        candidatos = [
            base + "." + str(i) for i in range(1, 255) if base + "." + str(i) != local_ip
        ]
        with ThreadPoolExecutor(max_workers=64) as pool:
            resultados = list(pool.map(
                lambda ip: ip if _puerto_abierto(ip, PUERTO_TCP, timeout) else None,
                candidatos,
            ))
        for r in resultados:
            if r:
                return r
        return None


    def buscar_carro(timeout=2.0):
        """Busca la Pico: broadcast UDP y, si no responde, escaneo de la subred."""
        ips = ips_locales()
        for ip in (ips or ["0.0.0.0"]):
            r = _udp_busqueda(ip, min(timeout, 1.5))
            if r:
                return r
        for ip in ips:
            r = escanear_subred(ip)
            if r:
                return r
        return None


    def diagnostico_red(host, puerto):
        """Líneas de texto que explican por qué la compu no llega a la Pico."""
        lineas = []
        ips = ips_locales(host)
        lineas.append("IP de esta compu: " + (", ".join(ips) if ips else "no detectada (¿sin WiFi?)"))
        lineas.append("IP de la Pico: " + host)
        base_pico = host.rsplit(".", 1)[0]
        if any(ip.rsplit(".", 1)[0] == base_pico for ip in ips):
            lineas.append("OK: la compu y la Pico están en la misma red (" + base_pico + ".x).")
        else:
            lineas.append(
                "PROBLEMA: la compu NO está en la red de la Pico (" + base_pico + ".x). "
                "Conecta la compu al mismo WiFi que la Pico (el de la pestaña WiFi)."
            )
        lineas.append("Probando el puerto %d de %s ..." % (puerto, host))
        try:
            with socket.create_connection((host, puerto), timeout=4):
                lineas.append("OK: el firmware responde. Ya puedes pulsar Conectar.")
        except ConnectionRefusedError:
            lineas.append(
                "La Pico está en la red, pero el firmware no está escuchando: "
                "reinicia la Pico (USB) o instala el firmware."
            )
        except OSError as e:
            lineas.append(
                "Sin respuesta de la Pico (" + str(e) + "). Causas típicas: la compu está "
                "en otra red, la red bloquea la comunicación entre dispositivos (red de "
                "invitados / aislamiento de clientes) o el firewall. Prueba en CMD: ping " + host
            )
        return lineas


    def extraer_firmware():
        """Devuelve solo la parte de la Pico de este mismo archivo."""
        with open(os.path.abspath(__file__), encoding="utf-8") as f:
            src = f.read()
        return src[src.index(MARCA_INI):]


    def _mpremote(args):
        cmd = [sys.executable, "-m", "mpremote"] + args
        r = subprocess.run(cmd, capture_output=True, text=True, encoding="utf-8",
                           errors="replace", timeout=120)
        return r.returncode, (r.stdout + r.stderr).strip()


    def _mpremote_auto(args, log):
        rc, out = _mpremote(args)
        if "No module named mpremote" in out:
            log("Instalando la herramienta mpremote (pip) ...")
            subprocess.run(
                [sys.executable, "-m", "pip", "install", "mpremote"],
                capture_output=True, text=True
            )
            rc, out = _mpremote(args)
        return rc, out


    def _escribir_wifi(carpeta, ssid, clave):
        ruta = os.path.join(carpeta, "wifi.json")
        with open(ruta, "w", encoding="utf-8") as f:
            json.dump({"ssid": ssid, "clave": clave}, f)
        return ruta


    def instalar_firmware(log, ssid="", clave=""):
        """Sube el firmware a la Pico como main.py (y wifi.json si hay SSID) y reinicia."""
        carpeta = tempfile.mkdtemp()
        ruta = os.path.join(carpeta, "main.py")
        with open(ruta, "w", encoding="utf-8") as f:
            f.write(extraer_firmware())
        args = ["connect", "auto", "fs", "cp", ruta, ":main.py"]
        if ssid:
            ruta_wifi = _escribir_wifi(carpeta, ssid, clave)
            args += ["+", "fs", "cp", ruta_wifi, ":wifi.json"]
        args += ["+", "reset"]
        return _mpremote_auto(args, log)


    def guardar_wifi(ssid, clave, log):
        """Guarda la red WiFi en la Pico (wifi.json) por USB y la reinicia."""
        carpeta = tempfile.mkdtemp()
        ruta_wifi = _escribir_wifi(carpeta, ssid, clave)
        args = ["connect", "auto", "fs", "cp", ruta_wifi, ":wifi.json", "+", "reset"]
        return _mpremote_auto(args, log)


    DIAG_WIFI = '''
import network, time
SSID = iPhone de Esteban
CLAVE = TNsS-inzE-OCr0-UzYL
EST = {3: "CONECTADO", 2: "conectado sin IP (DHCP)", 1: "conectando...", 0: "desconectado",
       -1: "fallo al conectar", -2: "no encuentra la red (nombre o banda)",
       -3: "contrasena incorrecta o seguridad no compatible"}
wlan = network.WLAN(network.STA_IF)
wlan.active(False)
time.sleep(1)
wlan.active(True)
try:
    wlan.config(pm=0xa11140)
except Exception:
    pass
print("=== Redes que ve la Pico ===")
visible = False
for r in sorted(wlan.scan(), key=lambda x: -x[3]):
    nombre = r[0].decode()
    marca = ""
    if nombre == SSID:
        marca = "   <-- tu red"
        visible = True
    print("  %-28s canal %2d  senal %4d  seg %s%s" % (nombre, r[2], r[3], r[4], marca))
if not visible:
    print("!! La red '%s' NO aparece: revisa el nombre exacto, que sea 2.4 GHz y la distancia." % SSID)
print("=== Conectando a %s ===" % SSID)
wlan.connect(SSID, CLAVE)
ultimo = None
for i in range(40):
    st = wlan.status()
    if st != ultimo:
        print("  %2d s  estado %d: %s" % (i, st, EST.get(st, "?")))
        ultimo = st
    if st == 3:
        break
    time.sleep(1)
st = wlan.status()
if wlan.isconnected():
    print("RESULTADO: CONECTADO. IP de la Pico: %s" % wlan.ifconfig()[0])
else:
    print("RESULTADO: NO CONECTO (estado %d: %s)" % (st, EST.get(st, "?")))
    if st == 1:
        print("Pista: se queda 'conectando' sin que el router responda. Pon la red en WPA2-PSK (AES), sin filtro de MAC, canal 1/6/11 a 20 MHz; o prueba con el hotspot del celular (2.4 GHz).")
    elif st == -3:
        print("Pista: revisa mayusculas/minusculas y espacios de la contrasena.")
    elif st == -2:
        print("Pista: la Pico no ve esa red; confirma que sea de 2.4 GHz.")
'''


    def probar_wifi(ssid, clave, log):
        """Corre un diagnostico de WiFi en la Pico por USB y devuelve lo que imprime."""
        carpeta = tempfile.mkdtemp()
        ruta = os.path.join(carpeta, "diag_wifi.py")
        with open(ruta, "w", encoding="utf-8") as f:
            f.write(DIAG_WIFI.replace("__SSID__", repr(ssid)).replace("__CLAVE__", repr(clave)))
        return _mpremote_auto(["connect", "auto", "run", ruta, "+", "reset"], log)


    def limpiar_host(texto):
        """Saca la IP (y el puerto si viene como ip:puerto) de lo que se pegue."""
        m = re.search(r"(\d{1,3}(?:\.\d{1,3}){3})(?::(\d{1,5}))?", texto)
        if not m:
            return None, None
        return m.group(1), (int(m.group(2)) if m.group(2) else None)


    def leer_respuestas(s):
        salida = b""
        try:
            while True:
                d = s.recv(1024)
                if not d:
                    break
                salida += d
        except socket.timeout:
            pass
        return salida.decode("utf-8", errors="replace")


    def pruebas_consola():
        """Programa de pruebas manuales por teclado (python main.py pruebas)."""
        print("Buscando el carro en la red ...")
        ip = buscar_carro() or input("No lo encontré. IP de la Pico: ").strip()
        s = socket.create_connection((ip, PUERTO_TCP), timeout=5)
        s.settimeout(0.7)
        print("Conectado a", ip)
        print("Escribe A B C (ej. 101), 'todas' o 'q'.")
        while True:
            texto = input("ABC> ").strip().lower().replace(" ", "")
            if texto in ("q", "salir"):
                break
            lista = [format(i, "03b") for i in range(8)] if texto == "todas" else [texto]
            for bits in lista:
                if len(bits) != 3 or any(c not in "01" for c in bits):
                    print("Entrada inválida: 3 bits, solo 0 y 1 (ej. 011)")
                    continue
                s.sendall(("abc " + bits + "\n").encode())
                time.sleep(0.2)
                print(leer_respuestas(s).strip())
                if texto == "todas":
                    time.sleep(1.2)
        s.close()


    # ---------------------------------------------------------------
    # INTERFAZ GRÁFICA
    # ---------------------------------------------------------------
    class ScrollFrame(ttk.Frame):
        """Marco con barra vertical: todo se alcanza a ver aunque la pantalla sea chica."""

        def __init__(self, parent, padding=0):
            super().__init__(parent)
            fondo = ttk.Style().lookup("TFrame", "background") or "#dcdad5"
            self.canvas = tk.Canvas(self, highlightthickness=0, bg=fondo)
            barra = ttk.Scrollbar(self, orient="vertical", command=self.canvas.yview)
            self.inner = ttk.Frame(self.canvas, padding=padding)
            self._ventana = self.canvas.create_window((0, 0), window=self.inner, anchor="nw")
            self.inner.bind(
                "<Configure>",
                lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all")),
            )
            self.canvas.bind(
                "<Configure>",
                lambda e: self.canvas.itemconfigure(self._ventana, width=e.width),
            )
            self.canvas.configure(yscrollcommand=barra.set)
            self.canvas.pack(side="left", fill="both", expand=True)
            barra.pack(side="right", fill="y")

        def _rueda(self, e):
            self.canvas.yview_scroll(-1 if e.delta > 0 else 1, "units")

        def habilitar_rueda(self):
            """Enlaza la rueda del mouse en el marco y en todo lo que contiene."""
            def enlazar(w):
                w.bind("<MouseWheel>", self._rueda, add="+")
                w.bind("<Button-4>", lambda e: self.canvas.yview_scroll(-1, "units"), add="+")
                w.bind("<Button-5>", lambda e: self.canvas.yview_scroll(1, "units"), add="+")
                for hijo in w.winfo_children():
                    enlazar(hijo)
            enlazar(self)


    class ControlFormulaE:
        def __init__(self, root):
            self.root = root
            self.root.title("Formula E CES TEC - Control")
            alto = min(900, self.root.winfo_screenheight() - 90)
            self.root.geometry(f"940x{alto}")
            self.root.minsize(820, 480)

            self.sock = None
            self.connected = False
            self.lock = threading.Lock()
            self.messages = queue.Queue()
            self.receive_thread = None
            self._photos = []

            self.host_var = tk.StringVar()
            self.port_var = tk.StringVar(value=str(PUERTO_TCP))
            self.speed_var = tk.IntVar(value=2)
            self.auto_ldr_var = tk.BooleanVar(value=True)
            self.monitor_ldr_var = tk.BooleanVar(value=False)
            self.dec_var = tk.BooleanVar(value=False)
            self.abc_var = tk.StringVar(value="000")
            self.ssid_var = tk.StringVar()
            self.pass_var = tk.StringVar()
            self.modo_celular = False
            self.reconectar = False        # True mientras el usuario quiere estar conectado
            self._reconectando = False
            self.ultimo_rx = 0.0           # ultima vez que llegó algo de la Pico
            self.umbral_var = tk.StringVar(value="1.0")
            self.voltaje_var = tk.StringVar(value="Voltaje: -- V")
            self.celular_var = tk.StringVar(value="")
            self.dec_luces_var = tk.StringVar(value="Luces encendidas: --")
            self.controles_pc = []       # marcos cuyos controles se bloquean en modo celular
            self.siempre_activos = []    # siguen activos en modo celular (parar, leer el LDR)

            self.light_vars = {
                "lf": tk.BooleanVar(value=False),
                "lb": tk.BooleanVar(value=False),
                "ll": tk.BooleanVar(value=False),
                "lr": tk.BooleanVar(value=False),
            }

            self._build_interface()
            self.host_var.trace_add("write", lambda *a: self._actualizar_info_celular())
            self.port_var.trace_add("write", lambda *a: self._actualizar_info_celular())
            self._actualizar_info_celular()
            self.root.after(100, self._process_messages)
            self.root.after(2000, self._latido)
            self.root.protocol("WM_DELETE_WINDOW", self.close)

        # ---------------------------------------------------------
        # INTERFAZ
        # ---------------------------------------------------------
        def _build_interface(self):
            style = ttk.Style()
            try:
                style.theme_use("clam")
            except tk.TclError:
                pass

            # --- Conexión (siempre visible arriba) ---
            connection = ttk.LabelFrame(self.root, text="Conexión WiFi (TCP)", padding=10)
            connection.pack(fill="x", padx=12, pady=(12, 6))

            ttk.Label(connection, text="IP de la Raspberry:").grid(
                row=0, column=0, sticky="w", padx=(0, 6)
            )
            ttk.Entry(connection, textvariable=self.host_var, width=18).grid(
                row=0, column=1, padx=5
            )
            ttk.Button(connection, text="Buscar carro", command=self.search_car).grid(
                row=0, column=2, padx=5
            )
            ttk.Label(connection, text="Puerto:").grid(
                row=0, column=3, sticky="w", padx=(12, 4)
            )
            ttk.Entry(connection, textvariable=self.port_var, width=7).grid(
                row=0, column=4, padx=5
            )
            self.connect_button = ttk.Button(
                connection, text="Conectar", command=self.connect
            )
            self.connect_button.grid(row=0, column=5, padx=(12, 5))
            self.disconnect_button = ttk.Button(
                connection, text="Desconectar", command=self.disconnect, state="disabled"
            )
            self.disconnect_button.grid(row=0, column=6, padx=5)

            self.status_label = ttk.Label(
                connection, text="● Desconectado", foreground="#b42318"
            )
            self.status_label.grid(row=1, column=0, columnspan=4, sticky="w", pady=(8, 0))

            self.install_button = ttk.Button(
                connection, text="Instalar firmware en la Pico (USB)",
                command=self.install_firmware,
            )
            self.install_button.grid(row=1, column=4, columnspan=3, sticky="e", pady=(8, 0))
            self.ip_label = ttk.Label(
                connection, text="IP de la Pico: (pulsa «Buscar carro»)",
                font=("Segoe UI", 11, "bold"), foreground="#555555",
            )
            self.ip_label.grid(row=2, column=0, columnspan=4, sticky="w", pady=(8, 0))
            ttk.Button(
                connection, text="Diagnosticar conexión", command=self.network_diag
            ).grid(row=2, column=4, columnspan=3, sticky="e", pady=(8, 0))

            # --- Modo celular (siempre visible) ---
            modo = ttk.LabelFrame(self.root, text="Modo celular (TCP client)", padding=10)
            modo.pack(fill="x", padx=12, pady=(0, 6))
            self.modo_btn = ttk.Button(
                modo, text="Pasar el control al celular", command=self.toggle_modo_celular
            )
            self.modo_btn.grid(row=0, column=0, sticky="w")
            self.modo_label = ttk.Label(
                modo, text="Control: PC", font=("Segoe UI", 11, "bold"), foreground="#16803c"
            )
            self.modo_label.grid(row=0, column=1, sticky="w", padx=(14, 0))
            ttk.Label(
                modo, textvariable=self.celular_var, foreground="#555555", wraplength=860,
                justify="left",
            ).grid(row=1, column=0, columnspan=2, sticky="w", pady=(8, 0))

            # --- Consola (siempre visible abajo) ---
            console_frame = ttk.LabelFrame(
                self.root, text="Mensajes de la Raspberry Pi Pico W", padding=8
            )
            console_frame.pack(side="bottom", fill="x", padx=12, pady=(6, 12))

            self.console = tk.Text(
                console_frame, height=5, wrap="word",
                state="disabled", font=("Consolas", 9),
            )
            self.console.pack(side="left", fill="both", expand=True)
            scrollbar = ttk.Scrollbar(
                console_frame, orient="vertical", command=self.console.yview
            )
            scrollbar.pack(side="right", fill="y")
            self.console.configure(yscrollcommand=scrollbar.set)
            ttk.Button(
                console_frame, text="Limpiar", command=self.clear_console
            ).pack(side="right", padx=(6, 0), anchor="n")

            # --- Pestañas ---
            notebook = ttk.Notebook(self.root)
            notebook.pack(fill="both", expand=True, padx=12, pady=6)

            sf_control = ScrollFrame(notebook, padding=10)
            notebook.add(sf_control, text="Control")
            sf_dec = ScrollFrame(notebook, padding=14)
            notebook.add(sf_dec, text="Decrementador")
            sf_wifi = ScrollFrame(notebook, padding=14)
            notebook.add(sf_wifi, text="WiFi")
            sf_about = ScrollFrame(notebook, padding=20)
            notebook.add(sf_about, text="About")

            self._build_control(sf_control.inner)
            self._build_decrementer(sf_dec.inner)
            self.controles_pc = [sf_control.inner, sf_dec.inner]
            self._build_wifi(sf_wifi.inner)
            self._build_about(sf_about.inner)
            for sf in (sf_control, sf_dec, sf_wifi, sf_about):
                sf.habilitar_rueda()

        def _build_control(self, main):
            top = ttk.Frame(main)
            top.pack(fill="x", pady=(0, 10))
            top.columnconfigure(0, weight=1)
            top.columnconfigure(1, weight=1)

            driving = ttk.LabelFrame(top, text="Control del vehículo", padding=12)
            driving.grid(row=0, column=0, sticky="nsew", padx=(0, 5))

            ttk.Label(
                driving,
                text="Los giros duran el tiempo configurado en MicroPython.",
                foreground="#555555", wraplength=300,
            ).pack(pady=(0, 8))

            directions = ttk.Frame(driving)
            directions.pack()

            ttk.Button(
                directions, text="↑ Avanzar", width=15,
                command=lambda: self.set_speed(abs(self.speed_var.get()) or 2),
            ).grid(row=0, column=1, padx=4, pady=4)
            ttk.Button(
                directions, text="← Izquierda", width=15,
                command=lambda: self.send_command("dir -1"),
            ).grid(row=1, column=0, padx=4, pady=4)
            ttk.Button(
                directions, text="Recto", width=15,
                command=lambda: self.send_command("dir 0"),
            ).grid(row=1, column=1, padx=4, pady=4)
            ttk.Button(
                directions, text="Derecha →", width=15,
                command=lambda: self.send_command("dir 1"),
            ).grid(row=1, column=2, padx=4, pady=4)
            ttk.Button(
                directions, text="↓ Retroceder", width=15,
                command=lambda: self.set_speed(-(abs(self.speed_var.get()) or 2)),
            ).grid(row=2, column=1, padx=4, pady=4)

            self.stop_button = ttk.Button(
                driving, text="DETENER VEHÍCULO", command=self.stop_vehicle
            )
            self.stop_button.pack(fill="x", pady=(12, 0))

            speed = ttk.LabelFrame(top, text="Potencia del motor", padding=12)
            speed.grid(row=0, column=1, sticky="nsew", padx=(5, 0))

            ttk.Label(speed, text="Selecciona un nivel entre 0 y 4.").pack(anchor="w")
            self.speed_value_label = ttk.Label(
                speed, text="Nivel: 2", font=("Segoe UI", 16, "bold")
            )
            self.speed_value_label.pack(pady=5)
            speed_scale = ttk.Scale(
                speed, from_=0, to=4, orient="horizontal",
                command=self._update_speed_label,
            )
            speed_scale.set(2)
            speed_scale.pack(fill="x", pady=8)
            ttk.Button(speed, text="Aplicar avance", command=self.apply_forward).pack(
                fill="x", pady=3
            )
            ttk.Button(speed, text="Aplicar retroceso", command=self.apply_reverse).pack(
                fill="x", pady=3
            )

            lower = ttk.Frame(main)
            lower.pack(fill="x", pady=(0, 10))
            lower.columnconfigure(0, weight=1)
            lower.columnconfigure(1, weight=1)

            lights = ttk.LabelFrame(lower, text="Iluminación", padding=10)
            lights.grid(row=0, column=0, sticky="nsew", padx=(0, 5))
            for i, (command, label) in enumerate([
                ("lf", "Luces delanteras"),
                ("lb", "Luces traseras"),
                ("ll", "Direccional izquierda"),
                ("lr", "Direccional derecha"),
            ]):
                ttk.Checkbutton(
                    lights, text=label, variable=self.light_vars[command],
                    command=lambda c=command: self.toggle_light(c),
                ).grid(row=i // 2, column=i % 2, sticky="w", padx=5, pady=5)

            sensor = ttk.LabelFrame(lower, text="Sensor de luz (LDR)", padding=10)
            sensor.grid(row=0, column=1, sticky="nsew", padx=(5, 0))
            ttk.Checkbutton(
                sensor, text="Iluminación automática",
                variable=self.auto_ldr_var, command=self.toggle_auto_ldr,
            ).pack(anchor="w", pady=3)
            monitor_chk = ttk.Checkbutton(
                sensor, text="Monitor continuo del LDR",
                variable=self.monitor_ldr_var, command=self.toggle_monitor,
            )
            monitor_chk.pack(anchor="w", pady=3)
            consulta_btn = ttk.Button(
                sensor, text="Consultar lectura del LDR",
                command=lambda: self.send_command("luz"),
            )
            consulta_btn.pack(fill="x", pady=(6, 0))
            ttk.Label(
                sensor, textvariable=self.voltaje_var, font=("Segoe UI", 12, "bold"),
            ).pack(anchor="w", pady=(8, 0))
            fila = ttk.Frame(sensor)
            fila.pack(fill="x", pady=(6, 0))
            ttk.Label(fila, text="Enciende bajo:").pack(side="left")
            ttk.Entry(fila, textvariable=self.umbral_var, width=6).pack(side="left", padx=4)
            ttk.Label(fila, text="V").pack(side="left")
            ttk.Button(fila, text="Aplicar", command=self.aplicar_umbral).pack(
                side="left", padx=6
            )
            self.siempre_activos += [self.stop_button, monitor_chk, consulta_btn]

            patterns = ttk.LabelFrame(main, text="Patrones automáticos", padding=10)
            patterns.pack(fill="x")
            for i, (label, command) in enumerate([
                ("Circle", "Circle"), ("Infinite", "Infinite"),
                ("ZigZag", "ZigZag"), ("Especial", "especial"),
            ]):
                ttk.Button(
                    patterns, text=label,
                    command=lambda c=command: self.run_pattern(c),
                ).grid(row=0, column=i, sticky="ew", padx=5, pady=4)
                patterns.columnconfigure(i, weight=1)
            ttk.Label(
                patterns,
                text="Los patrones pueden tardar varios segundos en terminar.",
                foreground="#8a4b08",
            ).grid(row=1, column=0, columnspan=4, sticky="w", padx=5, pady=(5, 0))

        def _build_decrementer(self, frame):
            auto = ttk.LabelFrame(frame, text="Modo con fotorresistencia", padding=10)
            auto.pack(fill="x", pady=(0, 12))
            ttk.Checkbutton(
                auto, text="Activar decrementador (entradas desde la fotorresistencia)",
                variable=self.dec_var, command=self.toggle_dec,
            ).pack(anchor="w")
            ttk.Label(
                auto,
                text="El LDR se divide en 8 rangos (0-7) → bits ABC → circuito → LED: "
                     "delanteras = X (MSB), direccionales = Y, traseras = Z (LSB). "
                     "Mientras está activo se desactiva la iluminación automática.",
                foreground="#555555", wraplength=820, justify="left",
            ).pack(anchor="w", pady=(4, 0))
            ttk.Label(
                auto, textvariable=self.dec_luces_var, font=("Segoe UI", 12, "bold"),
                foreground="#16803c", wraplength=820, justify="left",
            ).pack(anchor="w", pady=(8, 0))

            manual = ttk.LabelFrame(frame, text="Pruebas manuales (teclado)", padding=10)
            manual.pack(fill="x")
            ttk.Label(
                manual,
                text="Escribe los tres bits A B C (ej. 101) y pulsa Enter. El resultado "
                     "aparece en los LED y en la consola.",
                foreground="#555555", wraplength=820, justify="left",
            ).grid(row=0, column=0, columnspan=4, sticky="w", pady=(0, 8))

            entry = ttk.Entry(
                manual, textvariable=self.abc_var, width=6, font=("Consolas", 16),
                justify="center",
            )
            entry.grid(row=1, column=0, padx=(0, 10))
            entry.bind("<Return>", lambda e: self.send_abc())
            ttk.Button(manual, text="Enviar ABC", command=self.send_abc).grid(
                row=1, column=1, padx=5
            )
            ttk.Button(
                manual, text="Probar las 8 combinaciones", command=self.test_all
            ).grid(row=1, column=2, padx=5)

        def _build_wifi(self, frame):
            ttk.Label(
                frame, text="Red WiFi de la Pico", font=("Segoe UI", 12, "bold")
            ).pack(anchor="w")
            ttk.Label(
                frame,
                text="Aquí cambias la red a la que se conecta la Pico. Conéctala por USB "
                     "(cierra Thonny): la red se guarda en la Pico y se reinicia. La compu "
                     "debe estar en esa misma red para controlarla.",
                foreground="#555555", wraplength=820, justify="left",
            ).pack(anchor="w", pady=(4, 12))

            form = ttk.Frame(frame)
            form.pack(anchor="w")
            ttk.Label(form, text="Nombre de la red (SSID):").grid(
                row=0, column=0, sticky="w", pady=4
            )
            ttk.Entry(form, textvariable=self.ssid_var, width=32).grid(
                row=0, column=1, padx=8
            )
            ttk.Label(form, text="Contraseña:").grid(row=1, column=0, sticky="w", pady=4)
            self.pass_entry = ttk.Entry(
                form, textvariable=self.pass_var, width=32, show="•"
            )
            self.pass_entry.grid(row=1, column=1, padx=8)
            self.show_pass = tk.BooleanVar(value=False)
            ttk.Checkbutton(
                form, text="Mostrar", variable=self.show_pass,
                command=lambda: self.pass_entry.config(
                    show="" if self.show_pass.get() else "•"
                ),
            ).grid(row=1, column=2)

            self.wifi_button = ttk.Button(
                frame, text="Guardar WiFi en la Pico (USB)", command=self.save_wifi
            )
            self.wifi_button.pack(anchor="w", pady=(12, 4))
            self.test_button = ttk.Button(
                frame, text="Probar conexión WiFi (USB, muestra el diagnóstico)",
                command=self.test_wifi,
            )
            self.test_button.pack(anchor="w", pady=(0, 12))
            ttk.Label(
                frame,
                text="La Pico W solo se conecta a redes de 2.4 GHz. Si dejas el nombre "
                     "vacío al instalar el firmware, usa la red que trae por defecto.",
                foreground="#8a4b08", wraplength=820, justify="left",
            ).pack(anchor="w")

        def save_wifi(self):
            ssid = self.ssid_var.get().strip()
            if not ssid:
                messagebox.showwarning(
                    "Falta el nombre", "Escribe el nombre de la red WiFi (SSID)."
                )
                return
            if not messagebox.askyesno(
                "Guardar WiFi",
                "Conecta la Pico por USB y cierra Thonny u otros programas que usen el "
                "puerto.\n\nSe guardará la red «" + ssid + "» y la Pico se reiniciará. "
                "¿Continuar?",
            ):
                return
            clave = self.pass_var.get()
            self.wifi_button.config(state="disabled")
            self.test_button.config(state="disabled")
            self._log("Guardando la red WiFi en la Pico ...")

            def worker():
                try:
                    rc, out = guardar_wifi(
                        ssid, clave, lambda t: self.messages.put(("message", t))
                    )
                except Exception as e:  # noqa
                    rc, out = 1, str(e)
                if rc == 0:
                    self.messages.put((
                        "message",
                        "Red guardada. Esperando a que la Pico se conecte a «" + ssid + "» ...",
                    ))
                    for _ in range(8):
                        time.sleep(2.5)
                        ip = buscar_carro(2.0)
                        if ip:
                            self.messages.put(("found", ip))
                            break
                    else:
                        self.messages.put((
                            "message",
                            "No la encontré. Revisa el nombre y la contraseña (y que la "
                            "red sea de 2.4 GHz) y que la compu esté en esa red.",
                        ))
                else:
                    self.messages.put((
                        "message",
                        "No se pudo guardar. ¿Pico por USB, con MicroPython y el firmware "
                        "instalado, y Thonny cerrado?\n" + out,
                    ))
                self.messages.put(("wifi_done", ""))

            threading.Thread(target=worker, daemon=True).start()

        def test_wifi(self):
            ssid = self.ssid_var.get().strip()
            if not ssid:
                messagebox.showwarning(
                    "Falta el nombre",
                    "Escribe el nombre de la red y la contraseña que quieres probar.",
                )
                return
            clave = self.pass_var.get()
            self.wifi_button.config(state="disabled")
            self.test_button.config(state="disabled")
            self._log("Probando la conexión WiFi desde la Pico (por USB, ~1 minuto) ...")

            def worker():
                try:
                    rc, out = probar_wifi(
                        ssid, clave, lambda t: self.messages.put(("message", t))
                    )
                except Exception as e:  # noqa
                    rc, out = 1, str(e)
                for linea in out.splitlines():
                    if linea.strip():
                        self.messages.put(("message", linea))
                if rc != 0:
                    self.messages.put((
                        "message",
                        "No se pudo correr la prueba. ¿Pico por USB, con MicroPython, "
                        "y Thonny cerrado?",
                    ))
                self.messages.put(("wifi_done", ""))

            threading.Thread(target=worker, daemon=True).start()

        def _build_about(self, frame):
            ttk.Label(
                frame, text=NOMBRE_PROGRAMA, font=("Segoe UI", 18, "bold")
            ).pack(anchor="w")
            ttk.Label(frame, text="Versión " + VERSION, font=("Segoe UI", 10)).pack(
                anchor="w", pady=(0, 12)
            )

            ttk.Label(frame, text="Integrantes", font=("Segoe UI", 14, "bold")).pack(
                anchor="w"
            )
            fila = ttk.Frame(frame)
            fila.pack(anchor="w", pady=(6, 16))
            base = os.path.dirname(os.path.abspath(__file__))
            for col, (institucion, nombre, foto) in enumerate(INTEGRANTES):
                tarjeta = ttk.Frame(fila, padding=12, relief="groove", borderwidth=2)
                tarjeta.grid(row=0, column=col, padx=(0, 16), sticky="n")
                try:
                    if FOTOS_B64.get(foto):
                        img = tk.PhotoImage(data=FOTOS_B64[foto])
                    else:
                        img = tk.PhotoImage(file=os.path.join(base, foto))
                    factor = max(-(-img.width() // 220), -(-img.height() // 220), 1)
                    if factor > 1:
                        img = img.subsample(factor, factor)
                    self._photos.append(img)  # evita que Tk la borre
                    ttk.Label(tarjeta, image=img).pack()
                except tk.TclError:
                    ttk.Label(tarjeta, text="[sin foto]").pack(pady=40)
                ttk.Label(
                    tarjeta, text=nombre, font=("Segoe UI", 13, "bold"),
                    wraplength=240, justify="center",
                ).pack(pady=(10, 2))
                ttk.Label(
                    tarjeta, text=institucion, font=("Segoe UI", 10),
                    foreground="#555555", wraplength=240, justify="center",
                ).pack()

            ttk.Label(frame, text="Guía de uso", font=("Segoe UI", 14, "bold")).pack(
                anchor="w"
            )
            ttk.Label(frame, text=GUIA, justify="left", font=("Segoe UI", 10)).pack(
                anchor="w", pady=(4, 0)
            )

        # ---------------------------------------------------------
        # BÚSQUEDA E INSTALACIÓN
        # ---------------------------------------------------------
        def network_diag(self):
            host, puerto = limpiar_host(self.host_var.get())
            if not host:
                messagebox.showwarning(
                    "Falta la IP",
                    "Escribe la IP de la Pico (o pulsa «Buscar carro») para diagnosticar.",
                )
                return
            if self.connected:
                self._log("Ya estás conectado: la conexión funciona.")
                return
            if not puerto:
                try:
                    puerto = int(self.port_var.get())
                except ValueError:
                    puerto = PUERTO_TCP
            self._log("Diagnosticando la red ...")

            def worker():
                for linea in diagnostico_red(host, puerto):
                    self.messages.put(("message", linea))

            threading.Thread(target=worker, daemon=True).start()

        def search_car(self):
            if self.connected:
                self._log("Ya estás conectado a " + self.host_var.get() + ".")
                return
            self._log("Buscando el carro en la red ...")

            def worker():
                ip = buscar_carro(2.5)
                if ip:
                    self.messages.put(("found", ip))
                else:
                    self.messages.put((
                        "message",
                        "No se encontró el carro. Revisa que la compu y la Pico estén en el MISMO WiFi y "
                    "pulsa «Diagnosticar conexión» para ver el motivo.",
                    ))

            threading.Thread(target=worker, daemon=True).start()

        def install_firmware(self):
            if not messagebox.askyesno(
                "Instalar firmware",
                "Conecta la Pico por USB y cierra Thonny u otros programas que usen el "
                "puerto.\n\nSe copiará el firmware como main.py y la Pico se reiniciará. "
                "Si llenaste la pestaña WiFi, también se guarda esa red.\n\n"
                "¿Continuar?",
            ):
                return
            ssid = self.ssid_var.get().strip()
            clave = self.pass_var.get()
            self.install_button.config(state="disabled")
            self._log("Instalando firmware en la Pico (puede tardar unos segundos) ...")

            def worker():
                try:
                    rc, out = instalar_firmware(
                        lambda t: self.messages.put(("message", t)), ssid, clave
                    )
                except Exception as e:  # noqa
                    rc, out = 1, str(e)
                if rc == 0:
                    self.messages.put((
                        "message", "Firmware instalado. Esperando a que la Pico se conecte ..."
                    ))
                    for _ in range(8):
                        time.sleep(2.5)
                        ip = buscar_carro(2.0)
                        if ip:
                            self.messages.put(("found", ip))
                            break
                    else:
                        self.messages.put((
                            "message",
                            "No la encontré todavía. Pulsa «Buscar carro» en unos segundos.",
                        ))
                else:
                    self.messages.put((
                        "message",
                        "No se pudo instalar. ¿Pico conectada por USB, con MicroPython "
                        "instalado y Thonny cerrado?\n" + out,
                    ))
                self.messages.put(("install_done", ""))

            threading.Thread(target=worker, daemon=True).start()

        # ---------------------------------------------------------
        # CONEXIÓN TCP
        # ---------------------------------------------------------
        def connect(self):
            host, puerto_en_texto = limpiar_host(self.host_var.get())
            if not host:
                messagebox.showwarning(
                    "IP inválida",
                    "Escribe solo la IP de la Pico, por ejemplo 192.168.0.25, "
                    "o pulsa «Buscar carro».",
                )
                return
            self.host_var.set(host)
            if puerto_en_texto:
                self.port_var.set(str(puerto_en_texto))
            try:
                port = int(self.port_var.get())
                if not 1 <= port <= 65535:
                    raise ValueError
            except ValueError:
                messagebox.showerror(
                    "Puerto inválido", "El puerto debe ser un número entre 1 y 65535."
                )
                return

            self.connect_button.config(state="disabled")
            self._log(f"Conectando con {host}:{port} ...")
            threading.Thread(
                target=self._connect_worker, args=(host, port), daemon=True
            ).start()

        def _connect_worker(self, host, port, silencioso=False):
            """Conecta con la Pico. Devuelve True si lo logró. `silencioso`: sin ventanas de error."""
            new_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            new_socket.settimeout(5)
            try:
                new_socket.connect((host, port))
                new_socket.settimeout(1)
                with self.lock:
                    self.sock = new_socket
                    self.connected = True
                try:
                    new_socket.sendall(b"gui\n")   # la Pico distingue a la GUI de un celular
                except OSError:
                    pass
                self.ultimo_rx = time.time()
                self.messages.put(("connected", f"Conectado a {host}:{port}"))
                self.receive_thread = threading.Thread(
                    target=self._receive_loop, daemon=True
                )
                self.receive_thread.start()
                return True
            except OSError as error:
                new_socket.close()
                if silencioso:
                    return False
                if isinstance(error, ConnectionRefusedError):
                    detalle = (
                        "La Pico está en la red pero el firmware no está escuchando en "
                        "ese puerto. Instálalo con «Instalar firmware en la Pico (USB)»."
                    )
                elif isinstance(error, (socket.timeout, TimeoutError)):
                    detalle = (
                        "La Pico no responde en " + host + ". Revisa que esté encendida "
                        "con el firmware instalado y que la compu esté en la misma red "
                        "WiFi. Prueba «Buscar carro» para confirmar la IP."
                    )
                else:
                    detalle = str(error)
                self.messages.put(("connection_error", detalle))
                return False

        def _receive_loop(self):
            buffer = b""
            while self.connected:
                try:
                    with self.lock:
                        current_socket = self.sock
                    if current_socket is None:
                        break
                    data = current_socket.recv(1024)
                    if not data:
                        self.messages.put(
                            ("disconnected", "La Raspberry cerró la conexión.")
                        )
                        break
                    self.ultimo_rx = time.time()
                    buffer += data
                    while b"\n" in buffer:
                        line, buffer = buffer.split(b"\n", 1)
                        decoded = line.decode("utf-8", errors="replace").strip()
                        if decoded and decoded != "pong":     # 'pong' es solo el latido
                            self.messages.put(("message", decoded))
                except socket.timeout:
                    continue
                except OSError as error:
                    if self.connected:
                        self.messages.put(("disconnected", f"Conexión perdida: {error}"))
                    break

        def disconnect(self):
            self.reconectar = False        # lo pidió el usuario: no reconectar solo
            self._close_socket()
            self._set_disconnected("Desconectado por el usuario.")

        def _close_socket(self):
            with self.lock:
                current_socket = self.sock
                self.sock = None
                self.connected = False
            if current_socket is not None:
                try:
                    current_socket.shutdown(socket.SHUT_RDWR)
                except OSError:
                    pass
                try:
                    current_socket.close()
                except OSError:
                    pass

        def _set_disconnected(self, message):
            self.status_label.config(text="● Desconectado", foreground="#b42318")
            self.connect_button.config(state="normal")
            self.disconnect_button.config(state="disabled")
            self._aplicar_modo(False)     # sin conexión, los controles vuelven a la PC
            self._log(message)
            if self.reconectar:
                self._iniciar_reconexion()

        def _latido(self):
            """Cada 2 s manda 'ping'. Si la Pico se apaga o se cae el WiFi, el socket no avisa
            (no hay cierre limpio), así que sin esto la GUI diría «Conectado» para siempre."""
            try:
                if self.connected:
                    with self.lock:
                        s = self.sock
                    if s is not None:
                        if time.time() - self.ultimo_rx > 20:     # un patrón dura hasta 12 s
                            self.messages.put((
                                "disconnected",
                                "La Pico dejó de responder (¿sin energía o sin WiFi?).",
                            ))
                        else:
                            try:
                                s.sendall(b"ping\n")
                            except OSError:
                                pass
            finally:
                if self.root.winfo_exists():
                    self.root.after(2000, self._latido)

        def _iniciar_reconexion(self):
            if self._reconectando:
                return
            self._reconectando = True
            self.status_label.config(text="● Reconectando…", foreground="#b45309")
            self.connect_button.config(state="disabled")
            self.disconnect_button.config(state="normal")     # para cancelar
            self._log("Reconectando automáticamente (pulsa Desconectar para cancelar) ...")
            threading.Thread(target=self._reconexion_worker, daemon=True).start()

        def _reconexion_worker(self):
            """Reintenta con la misma IP y, si no responde, busca la Pico otra vez
            (con el hotspot del celular la IP puede cambiar al reiniciarse la Pico)."""
            intento = 0
            try:
                while self.reconectar and not self.connected:
                    intento += 1
                    host = self.host_var.get().strip()
                    try:
                        port = int(self.port_var.get())
                    except ValueError:
                        port = PUERTO_TCP
                    if host and self._connect_worker(host, port, silencioso=True):
                        return
                    ip = buscar_carro(1.5)
                    if ip:
                        if ip != host:
                            self.messages.put(("found", ip))
                        if self._connect_worker(ip, port, silencioso=True):
                            return
                    if intento % 5 == 0:
                        self.messages.put((
                            "message", f"Sigo buscando la Pico (intento {intento}) ..."
                        ))
                    time.sleep(2)
            finally:
                self._reconectando = False

        # ---------------------------------------------------------
        # ENVÍO DE COMANDOS
        # ---------------------------------------------------------
        def send_command(self, command):
            with self.lock:
                current_socket = self.sock
                is_connected = self.connected
            if not is_connected or current_socket is None:
                self._log("No se envió el comando: primero conecta con la Raspberry.")
                return
            try:
                current_socket.sendall((command + "\n").encode("utf-8"))
                self._log(f">> {command}")
            except OSError as error:
                self._log(f"Error al enviar el comando: {error}")
                self._close_socket()
                self._set_disconnected("Se perdió la conexión.")

        def set_speed(self, value):
            value = max(-4, min(4, int(value)))
            self.speed_var.set(abs(value))
            self.send_command(f"pwm {value}")

        def apply_forward(self):
            self.set_speed(self.speed_var.get())

        def apply_reverse(self):
            self.set_speed(-self.speed_var.get())

        def stop_vehicle(self):
            self.send_command("stop")

        def _update_speed_label(self, value):
            level = round(float(value))
            self.speed_var.set(level)
            self.speed_value_label.config(text=f"Nivel: {level}")

        def toggle_light(self, command):
            value = 1 if self.light_vars[command].get() else 0
            self.send_command(f"{command} {value}")

        def toggle_auto_ldr(self):
            value = 1 if self.auto_ldr_var.get() else 0
            if value and self.dec_var.get():
                # la iluminación automática y el decrementador no pueden coexistir
                self.dec_var.set(False)
                self.send_command("dec 0")
            self.send_command(f"ldr {value}")

        def toggle_dec(self):
            value = 1 if self.dec_var.get() else 0
            if value:
                self.auto_ldr_var.set(False)  # la Pico desactiva el modo automático
            self.send_command(f"dec {value}")

        def toggle_monitor(self):
            value = 1 if self.monitor_ldr_var.get() else 0
            self.send_command(f"luz {value}")

        def _valid_abc(self):
            bits = self.abc_var.get().replace(" ", "")
            if len(bits) != 3 or any(c not in "01" for c in bits):
                messagebox.showwarning(
                    "Entrada inválida", "Escribe exactamente 3 bits, solo 0 y 1 (ej. 101)."
                )
                return None
            return bits

        def send_abc(self):
            bits = self._valid_abc()
            if bits:
                self.dec_var.set(False)       # la prueba manual desactiva el modo LDR
                self.auto_ldr_var.set(False)
                self.send_command(f"abc {bits}")

        def test_all(self):
            self.dec_var.set(False)
            self.auto_ldr_var.set(False)
            for i in range(8):
                self.root.after(
                    i * 1500, lambda b=format(i, "03b"): self.send_command("abc " + b)
                )

        # ---------------------------------------------------------
        # MODO CELULAR Y SINCRONIZACIÓN CON LA PICO
        # ---------------------------------------------------------
        def toggle_modo_celular(self):
            if not self.connected:
                messagebox.showinfo(
                    "Modo celular",
                    "Primero conecta esta ventana con la Pico (botón Conectar). La GUI "
                    "queda como panel de monitoreo y el celular manda los comandos.",
                )
                return
            # el cambio se refleja cuando la Pico confirma con «MODO celular / MODO pc»
            self.send_command("modo pc" if self.modo_celular else "modo celular")

        def _habilitar_controles(self, activo):
            estado = ["!disabled"] if activo else ["disabled"]

            def recorrer(w):
                for hijo in w.winfo_children():
                    if isinstance(hijo, (ttk.Button, ttk.Checkbutton, ttk.Entry, ttk.Scale)):
                        hijo.state(estado)
                    recorrer(hijo)

            for marco in self.controles_pc:
                recorrer(marco)
            if not activo:
                for w in self.siempre_activos:
                    w.state(["!disabled"])

        def _aplicar_modo(self, celular):
            self.modo_celular = celular
            if celular:
                self.modo_label.config(
                    text="Control: CELULAR  (esta ventana solo muestra lo que pasa)",
                    foreground="#b45309",
                )
                self.modo_btn.config(text="Recuperar el control en la PC")
            else:
                self.modo_label.config(text="Control: PC", foreground="#16803c")
                self.modo_btn.config(text="Pasar el control al celular")
            self._habilitar_controles(not celular)

        def _actualizar_info_celular(self):
            host = self.host_var.get().strip() or "(IP de la Pico)"
            puerto = self.port_var.get().strip() or str(PUERTO_TCP)
            self.celular_var.set(
                f"En el celular (app de «TCP client»): IP {host}, puerto {puerto}. "
                "Comandos de ejemplo: pwm 2 · dir -1 · lf 1 · stop · Circle · ayuda. "
                "Para tomar el control desde el celular envía: modo celular"
            )

        def aplicar_umbral(self):
            try:
                valor = float(self.umbral_var.get().strip().replace(",", "."))
                if not 0.1 <= valor <= 3.0:
                    raise ValueError
            except ValueError:
                messagebox.showwarning(
                    "Umbral inválido", "Escribe un voltaje entre 0.1 y 3.0 (ej. 1.5)."
                )
                return
            self.send_command(f"umbral {valor:.2f}")

        def _sync_estado(self, texto):
            """Refleja en la GUI lo que la Pico informa (también lo que hace el celular)."""
            t = texto.strip()
            m = re.search(r"encendidas:\s*(.*)$", t)
            if m:
                self.dec_luces_var.set("Luces encendidas: " + m.group(1).strip())
            m = re.match(r"^MODO\s+(celular|pc)\b", t)
            if m:
                self._aplicar_modo(m.group(1) == "celular")
                return
            m = re.search(r"Voltaje:\s*([0-9.]+)\s*V", t)
            if m:
                self.voltaje_var.set(f"Voltaje: {m.group(1)} V")
                return
            m = re.match(r"^ldr -> ([01])$", t)
            if m:
                self.auto_ldr_var.set(m.group(1) == "1")
                return
            m = re.match(r"^dec -> ([01])$", t)
            if m:
                self.dec_var.set(m.group(1) == "1")
                if m.group(1) == "1":
                    self.auto_ldr_var.set(False)
                return
            m = re.match(r"^(lf|lb|ll|lr) -> ([01])$", t)
            if m:
                self.light_vars[m.group(1)].set(m.group(2) == "1")
                return
            if t.startswith("LDR:") and "luces" in t:
                encendidas = t.rstrip().endswith("ON")
                self.light_vars["lf"].set(encendidas)
                self.light_vars["lb"].set(encendidas)

        def run_pattern(self, command):
            answer = messagebox.askyesno(
                "Confirmar patrón",
                f"¿Quieres ejecutar el patrón {command}?\n\n"
                "El vehículo se moverá automáticamente durante varios segundos. "
                "Asegúrate de que tenga espacio libre.",
            )
            if answer:
                self.send_command(command)

        # ---------------------------------------------------------
        # CONSOLA Y CIERRE
        # ---------------------------------------------------------
        def _log(self, message):
            timestamp = datetime.now().strftime("%H:%M:%S")
            self.console.config(state="normal")
            self.console.insert("end", f"[{timestamp}] {message}\n")
            self.console.see("end")
            self.console.config(state="disabled")

        def clear_console(self):
            self.console.config(state="normal")
            self.console.delete("1.0", "end")
            self.console.config(state="disabled")

        def _process_messages(self):
            try:
                while True:
                    kind, message = self.messages.get_nowait()

                    if kind == "connected":
                        self.ip_label.config(
                            text="IP de la Pico: " + self.host_var.get().strip(), foreground="#16803c"
                        )
                        self.status_label.config(text="● Conectado", foreground="#16803c")
                        self.reconectar = True      # si se cae la conexión, reconecta sola
                        self.disconnect_button.config(state="normal")
                        self.connect_button.config(state="disabled")
                        self._log(message)

                    elif kind == "connection_error":
                        self.connect_button.config(state="normal")
                        self.status_label.config(
                            text="● Error de conexión", foreground="#b42318"
                        )
                        self._log(f"No se pudo conectar: {message}")
                        messagebox.showerror(
                            "Error de conexión",
                            f"No se pudo conectar con la Raspberry Pi Pico W.\n\n{message}",
                        )

                    elif kind == "disconnected":
                        self._close_socket()
                        self._set_disconnected(message)

                    elif kind == "found":
                        self.host_var.set(message)
                        self.ip_label.config(text="IP de la Pico: " + message, foreground="#16803c")
                        self._log(f"Carro encontrado en {message}")

                    elif kind == "install_done":
                        self.install_button.config(state="normal")

                    elif kind == "wifi_done":
                        self.wifi_button.config(state="normal")
                        self.test_button.config(state="normal")

                    elif kind == "message":
                        self._log(message)
                        self._sync_estado(message)

            except queue.Empty:
                pass

            if self.root.winfo_exists():
                self.root.after(100, self._process_messages)

        def close(self):
            self.reconectar = False
            if self.connected:
                try:
                    self.send_command("stop")
                except Exception:
                    pass
            self._close_socket()
            self.root.destroy()


    if len(sys.argv) > 1 and sys.argv[1] == "pruebas":
        pruebas_consola()
    else:
        ventana = tk.Tk()
        app = ControlFormulaE(ventana)
        ventana.mainloop()

    sys.exit(0)


# ==== INICIO FIRMWARE ====
# -*- coding: utf-8 -*-
# Proyecto Formula E CES Tec - Parte I + Parte II
# Raspberry Pi Pico W / 2W + 74LS164 + L298N  (MicroPython)
#
# Todo en un solo archivo: comandos de la Tabla 1, servidor WiFi (TCP) para la
# interfaz de Python y decrementador.
#
# Comandos (por WiFi desde la GUI, o por consola si el WiFi falla):
#   pwm  [-4..4]   dir  [-1,0,1,2]   lf/lb/ll/lr [1,0]   ldr [1,0]
#   Circle | Infinite | ZigZag | especial | stop | test
#   luz [1|0]      lectura / monitor del LDR
#   umbral <V>     voltaje bajo el cual el LDR enciende las luces (ej. umbral 1.5)
#   ldrinv [1|0]   invierte el LDR si tu divisor esta al reves
#   modo celular | modo pc   quien manda: el celular (TCP client) o la GUI
#   ayuda          lista de comandos
#   dec [1|0]      decrementador: el LDR (8 rangos, 0-7) -> ABC -> circuito -> LED
#
# Los LED muestran la salida del decrementador asi:
#   X (MSB) -> luces delanteras | Y -> direccionales | Z (LSB) -> luces traseras

from machine import Pin, PWM, ADC, Timer
import time

# ---------------------------------------------------------------
# WIFI
# ---------------------------------------------------------------
WIFI_SSID = "ApartasCM"
WIFI_CLAVE = "ApartamentosCM2332"
PUERTO = 8001
PUERTO_BUSQUEDA = 8002   # UDP: la GUI encuentra la IP de la Pico sola
USAR_WIFI = True          # False = solo consola de Thonny

# ---------------------------------------------------------------
# PINES
# ---------------------------------------------------------------
PIN_DATO = 15      # A y B del 74LS164 (unidos)
PIN_CLK = 14       # CLK del 74LS164
PIN_ENA = 16
PIN_IN1 = 17
PIN_IN2 = 18
PIN_ENB = 19
PIN_IN3 = 20
PIN_IN4 = 21
PIN_LDR = 27       # ADC1

# Decrementador: entradas A,B,C del circuito (salidas de la Pico)
PIN_DEC_A = 0
PIN_DEC_B = 1
PIN_DEC_C = 5
# Salidas X,Y,Z del circuito (entradas de la Pico). SUPUESTO: GP2, GP3, GP4.
PIN_DEC_X = 2
PIN_DEC_Y = 3
PIN_DEC_Z = 4

# False: la Pico calcula X,Y,Z con tus ecuaciones (sirve para probar sin el circuito).
# True : la Pico LEE X,Y,Z del circuito fisico (ponlo en True para la defensa).
LEER_SALIDAS_FISICAS = False

# Rango util del LDR (lectura cruda 0..65535) que se divide en 8 intervalos.
# Calibralo con el comando 'luz' (oscuridad total y mucha luz).
LDR_RAW_MIN = 0
LDR_RAW_MAX = 65535

# ---------------------------------------------------------------
# PARAMETROS A CALIBRAR CON TU CARRO
# ---------------------------------------------------------------
FREQ_PWM = 1000
DUTY_MIN = 32000
DUTY_MAX = 65535
DUTY_GIRO = 50000
TIEMPO_DIR = 1.5
TIEMPO_CIRCULO = 6.0
TIEMPO_ZIGZAG = 0.8
REPETICIONES_ZIGZAG = 4
NIVEL_PATRON = 2
CONVERSION_VOLTIOS = 3.3 / 65535
V_OSCURO = 1.0
V_CLARO = 1.3
HISTERESIS_LDR = 0.3   # separacion entre 'enciende' y 'apaga' (evita parpadeo)
MAX_CLIENTES = 3       # GUI + celular + uno extra
LDR_INVERTIDO = False
INVERTIR_DIRECCIONALES = True
INVERTIR_TRACCION = True
INVERTIR_GIRO = True   # True: corrige izquierda/derecha del motor de direccion

# ---------------------------------------------------------------
# PRINT: sale por Thonny/consola Y por el socket hacia la GUI
# ---------------------------------------------------------------
_print = print
_clientes = []          # clientes TCP conectados (GUI de la compu, celulares, etc.)
_modo_celular = False   # False: manda la GUI de la compu | True: manda el celular


def _enviar(c, texto):
    """Manda una linea a UN cliente."""
    try:
        c.sock.send((texto + "\n").encode())
    except Exception:
        pass


def _difundir(texto, excepto=None):
    """Manda una linea a todos los clientes (menos `excepto`)."""
    for c in _clientes:
        if c is not excepto:
            _enviar(c, texto)


def print(*args):
    texto = " ".join(str(a) for a in args)
    _print(texto)
    _difundir(texto)


# ---------------------------------------------------------------
# HARDWARE
# ---------------------------------------------------------------
dato = Pin(PIN_DATO, Pin.OUT, value=0)
clk = Pin(PIN_CLK, Pin.OUT, value=0)

ena = Pin(PIN_ENA, Pin.OUT, value=1)
enb = Pin(PIN_ENB, Pin.OUT, value=1)
in1 = PWM(Pin(PIN_IN1)); in1.freq(FREQ_PWM); in1.duty_u16(0)
in2 = PWM(Pin(PIN_IN2)); in2.freq(FREQ_PWM); in2.duty_u16(0)
in3 = PWM(Pin(PIN_IN3)); in3.freq(FREQ_PWM); in3.duty_u16(0)
in4 = PWM(Pin(PIN_IN4)); in4.freq(FREQ_PWM); in4.duty_u16(0)

ldr_adc = ADC(Pin(PIN_LDR))

dec_a = Pin(PIN_DEC_A, Pin.OUT, value=0)
dec_b = Pin(PIN_DEC_B, Pin.OUT, value=0)
dec_c = Pin(PIN_DEC_C, Pin.OUT, value=0)
if LEER_SALIDAS_FISICAS:
    # pull-down: una salida sin conectar lee 0
    dec_x = Pin(PIN_DEC_X, Pin.IN, Pin.PULL_DOWN)
    dec_y = Pin(PIN_DEC_Y, Pin.IN, Pin.PULL_DOWN)
    dec_z = Pin(PIN_DEC_Z, Pin.IN, Pin.PULL_DOWN)

# ---------------------------------------------------------------
# ESTADO
# ---------------------------------------------------------------
luces = {"lr": 0, "lf": 0, "lb": 0, "ll": 0}
modo_ldr = True
monitor_ldr = False
nivel_actual = 0
_en_uso = False
_timer = None
_ip = "?"          # IP de la Pico en el WiFi
_wlan = None       # interfaz WiFi (para detectar que se cayo la conexion)
modo_dec = False
_dec_ultimo = -1
_ldr_oscuro = None   # ultima decision del LDR: True oscuro / False claro / None sin decidir
_en_patron = False   # True mientras corre 'especial' (el LDR no toca las luces)
_ult_error = None


# ---------------------------------------------------------------
# DECREMENTADOR
# ---------------------------------------------------------------
def leer_ldr_raw(n=8):
    """Promedia n lecturas: el ADC de la Pico W es ruidoso con el WiFi encendido."""
    total = 0
    for _ in range(n):
        total += ldr_adc.read_u16()
    return total // n


def raw_a_voltaje(raw):
    v = raw * CONVERSION_VOLTIOS
    return 3.3 - v if LDR_INVERTIDO else v


def dec_ldr_a_valor(raw):
    """Divide el rango del LDR en 8 intervalos iguales -> entero 0..7."""
    if raw < LDR_RAW_MIN:
        raw = LDR_RAW_MIN
    if raw > LDR_RAW_MAX:
        raw = LDR_RAW_MAX
    return min(7, (raw - LDR_RAW_MIN) * 8 // (LDR_RAW_MAX - LDR_RAW_MIN + 1))


def dec_software(v):
    """Tus ecuaciones: X=A'C+C'(A xor B)  Y=B xnor C  Z=C'  -> entero 0..7."""
    a = (v >> 2) & 1
    b = (v >> 1) & 1
    c = v & 1
    x = ((1 - a) & c) | ((1 - c) & (a ^ b))
    y = 1 - (b ^ c)
    z = 1 - c
    return (x << 2) | (y << 1) | z


def dec_enviar(v):
    """Pone v (0..7) en A,B,C. Devuelve X,Y,Z (del circuito o por software)."""
    dec_a.value((v >> 2) & 1)
    dec_b.value((v >> 1) & 1)
    dec_c.value(v & 1)
    if LEER_SALIDAS_FISICAS:
        time.sleep_ms(2)
        return (dec_x.value() << 2) | (dec_y.value() << 1) | dec_z.value()
    return dec_software(v)


def nombres_luces(s):
    """XYZ (0..7) -> texto con las luces que quedan encendidas."""
    n = []
    if (s >> 2) & 1:
        n.append("delanteras (QB,QC)")
    if (s >> 1) & 1:
        n.append("direccionales (QA,QF)")
    if s & 1:
        n.append("traseras (QD,QE)")
    return ", ".join(n) if n else "ninguna"


def revisar_dec():
    global _dec_ultimo
    if not modo_dec or _en_uso:
        return
    v = dec_ldr_a_valor(leer_ldr_raw())
    s = dec_enviar(v)
    luces["lf"] = (s >> 2) & 1
    luces["ll"] = (s >> 1) & 1
    luces["lr"] = (s >> 1) & 1
    luces["lb"] = s & 1
    actualizar_luces()
    if v != _dec_ultimo:
        _dec_ultimo = v
        print("DEC: rango LDR={}  ABC={:03b}  ->  XYZ={:03b}  | encendidas: {}".format(
            v, v, s, nombres_luces(s)))


def cmd_dec(valor):
    global modo_dec, modo_ldr, _dec_ultimo
    if valor not in (0, 1):
        print("dec solo acepta 1 o 0")
        return
    if valor == 1:
        modo_ldr = False          # el auto-LDR choca con el decrementador
        _dec_ultimo = -1
        modo_dec = True
    else:
        modo_dec = False
        dec_enviar(0)
        for k in luces:
            luces[k] = 0
        actualizar_luces()
    print("dec ->", valor)


def cmd_abc(texto):
    """Prueba manual: abc 101 -> manda A,B,C al circuito y muestra X,Y,Z en los LED."""
    global modo_dec, modo_ldr
    if len(texto) != 3 or any(ch not in "01" for ch in texto):
        print("Uso: abc 101 (tres bits, solo 0 y 1)")
        return
    v = int(texto, 2)
    modo_dec = False
    modo_ldr = False
    s = dec_enviar(v)
    luces["lf"] = (s >> 2) & 1
    luces["ll"] = (s >> 1) & 1
    luces["lr"] = (s >> 1) & 1
    luces["lb"] = s & 1
    actualizar_luces()
    esp = dec_software(v)
    print("ABC={:03b}  ->  XYZ={:03b}  (tabla: {:03b})  {}  | encendidas: {}".format(
        v, s, esp, "OK" if s == esp else "DIFIERE", nombres_luces(s)))


# ---------------------------------------------------------------
# LUCES (74LS164)
# ---------------------------------------------------------------
def actualizar_luces():
    """Envia 6 bits al 74LS164 en orden QF, QE, QD, QC, QB, QA."""
    if INVERTIR_DIRECCIONALES:
        qa = luces["ll"]
        qf = luces["lr"]
    else:
        qa = luces["lr"]
        qf = luces["ll"]
    qb = luces["lf"]
    qc = luces["lf"]
    atras = 1 if (luces["lb"] or nivel_actual < 0) else 0
    qd = atras
    qe = atras
    global _en_uso
    _en_uso = True
    for bit in (qf, qe, qd, qc, qb, qa):
        dato.value(bit)
        clk.value(1)
        clk.value(0)
    _en_uso = False


def poner_luz(nombre, valor):
    luces[nombre] = 1 if valor else 0
    actualizar_luces()


def revisar_ldr():
    """Iluminacion automatica: oscuro -> enciende delanteras y traseras; claro -> apaga.

    Las luces SIGUEN al sensor (no dependen de en que estado estaban antes) y hay
    histeresis: enciende bajo V_OSCURO y apaga sobre V_CLARO."""
    global _ldr_oscuro
    if not modo_ldr or modo_dec or _en_uso or _en_patron:
        return
    v = raw_a_voltaje(leer_ldr_raw())
    if v < V_OSCURO:
        oscuro = True
    elif v > V_CLARO:
        oscuro = False
    elif _ldr_oscuro is None:
        oscuro = v < (V_OSCURO + V_CLARO) / 2   # primera lectura dentro de la histeresis
    else:
        oscuro = _ldr_oscuro
    _ldr_oscuro = oscuro
    d = 1 if oscuro else 0
    if luces["lf"] != d or luces["lb"] != d:
        luces["lf"] = d
        luces["lb"] = d
        actualizar_luces()
        print("LDR: {} ({:.2f} V) -> luces {}".format(
            "oscuro" if d else "claro", v, "ON" if d else "OFF"))


def esperar(segundos):
    fin = time.ticks_add(time.ticks_ms(), int(segundos * 1000))
    while time.ticks_diff(fin, time.ticks_ms()) > 0:
        revisar_ldr()
        time.sleep_ms(30)


# ---------------------------------------------------------------
# MOTORES (L298N)
# ---------------------------------------------------------------
def duty_nivel(nivel):
    n = abs(nivel)
    if n == 0:
        return 0
    return DUTY_MIN + (n - 1) * (DUTY_MAX - DUTY_MIN) // 3


def _motor(pa, pb, duty, sentido):
    if sentido > 0:
        pb.duty_u16(0)
        pa.duty_u16(duty)
    elif sentido < 0:
        pa.duty_u16(0)
        pb.duty_u16(duty)
    else:
        pa.duty_u16(0)
        pb.duty_u16(0)


def traccion(nivel):
    global nivel_actual
    sentido = -nivel if INVERTIR_TRACCION else nivel
    if nivel != 0 and nivel_actual == 0:
        _motor(in1, in2, DUTY_MAX, sentido)
        time.sleep_ms(120)
    nivel_actual = nivel
    actualizar_luces()
    _motor(in1, in2, duty_nivel(nivel), sentido)


def direccion(sentido):
    _motor(in4, in3, DUTY_GIRO, -sentido if INVERTIR_GIRO else sentido)


def detener_todo():
    traccion(0)
    direccion(0)


# ---------------------------------------------------------------
# COMANDOS DE MANDO
# ---------------------------------------------------------------
def cmd_pwm(valor):
    if valor < -4 or valor > 4:
        print("pwm fuera de rango [-4, 4]")
        return
    traccion(valor)
    print("pwm ->", valor)


def cmd_dir(valor):
    if valor not in (-1, 0, 1, 2):
        print("dir fuera de rango [-1, 0, 1, 2]")
        return
    if valor == 0:
        direccion(0)
        print("dir -> recto")
    elif valor == 2:
        direccion(0)
        nivel = -abs(nivel_actual) if nivel_actual != 0 else -NIVEL_PATRON
        traccion(nivel)
        esperar(TIEMPO_DIR)
        traccion(0)
        print("dir -> retroceder")
    else:
        direccion(valor)
        poner_luz("ll" if valor == -1 else "lr", 1)
        esperar(TIEMPO_DIR)
        direccion(0)
        poner_luz("ll" if valor == -1 else "lr", 0)
        print("dir ->", "izquierda" if valor == -1 else "derecha")


def cmd_luz(nombre, valor):
    global modo_ldr
    if valor not in (0, 1):
        print(nombre, "solo acepta 1 o 0")
        return
    if modo_ldr and nombre in ("lf", "lb"):
        modo_ldr = False          # control manual de luces: se apaga la iluminacion automatica
        print("ldr -> 0")
    poner_luz(nombre, valor)
    print(nombre, "->", valor)


def cmd_luz_lectura():
    raw = leer_ldr_raw()
    print("valor ADC: {:5d} / Voltaje: {:.2f} V / auto: {} / enciende bajo {:.2f} V, apaga sobre {:.2f} V".format(
        raw, raw_a_voltaje(raw), "ON" if modo_ldr else "OFF", V_OSCURO, V_CLARO))


def cmd_monitor(valor):
    global monitor_ldr
    monitor_ldr = (valor == 1)
    if monitor_ldr:
        print("Monitor LDR activo. Escribe 'luz 0' para detenerlo.")
    else:
        print("Monitor LDR detenido.")


def cmd_ldr(valor):
    global modo_ldr, modo_dec, _ldr_oscuro
    if valor not in (0, 1):
        print("ldr solo acepta 1 o 0")
        return
    if valor == 1 and modo_dec:
        modo_dec = False              # la iluminacion automatica manda: se apaga el decrementador
        dec_enviar(0)
        for k in luces:
            luces[k] = 0              # incluye las direccionales que dejo el decrementador
        actualizar_luces()
        print("dec -> 0")
    modo_ldr = bool(valor)
    _ldr_oscuro = None            # decide de cero con la lectura actual
    if modo_ldr:
        cmd_luz_lectura()
        print("LDR activo: enciende bajo {} V, apaga sobre {} V".format(V_OSCURO, V_CLARO))
        revisar_ldr()
    else:
        luces["lf"] = 0
        luces["lb"] = 0
        actualizar_luces()
    print("ldr ->", valor)


def cmd_umbral(texto):
    """umbral 1.5 -> las luces se encienden bajo 1.5 V y se apagan sobre 1.8 V."""
    global V_OSCURO, V_CLARO, _ldr_oscuro
    try:
        v = float(texto.replace(",", "."))      # en el celular el teclado suele dar coma
    except ValueError:
        print("Uso: umbral 1.5  (voltaje bajo el cual se encienden las luces)")
        return
    if v < 0.1 or v > 3.0:
        print("umbral fuera de rango [0.1, 3.0] V")
        return
    V_OSCURO = v
    V_CLARO = min(3.3, v + HISTERESIS_LDR)
    _ldr_oscuro = None
    print("umbral -> enciende bajo {:.2f} V, apaga sobre {:.2f} V".format(V_OSCURO, V_CLARO))


def cmd_ldrinv(valor):
    global LDR_INVERTIDO, _ldr_oscuro
    LDR_INVERTIDO = bool(valor)
    _ldr_oscuro = None
    print("ldrinv ->", valor)


def cmd_modo(arg):
    """modo celular | modo pc | modo (sin argumento: informa el modo actual)."""
    global _modo_celular
    if arg in ("celular", "cel", "movil", "telefono", "phone"):
        nuevo = True
    elif arg in ("pc", "gui", "compu"):
        nuevo = False
    elif arg == "":
        nuevo = _modo_celular
    else:
        print("Uso: modo celular | modo pc")
        return
    if nuevo != _modo_celular:
        detener_todo()            # al pasar el control, el carro se detiene por seguridad
    _modo_celular = nuevo
    print("MODO", "celular" if _modo_celular else "pc")


def cmd_ayuda():
    print("Comandos: pwm [-4..4] | dir [-1,0,1,2] | lf lb ll lr [1,0] | stop | test")
    print("Patrones: Circle | Infinite | ZigZag | especial")
    print("LDR: ldr [1,0] | luz | luz [1,0] | umbral <V> | ldrinv [1,0] | dec [1,0] | abc 101")
    print("Control: modo celular | modo pc | ip")


def cmd_test():
    print("test: traccion adelante");  traccion(4);  esperar(1); traccion(0)
    esperar(0.5)
    print("test: traccion atras");     traccion(-4); esperar(1); traccion(0)
    esperar(0.5)
    print("test: direccion izquierda"); direccion(-1); esperar(1); direccion(0)
    esperar(0.5)
    print("test: direccion derecha");   direccion(1);  esperar(1); direccion(0)
    print("test: fin")


# ---------------------------------------------------------------
# PATRONES
# ---------------------------------------------------------------
def _nivel_patron():
    return abs(nivel_actual) if nivel_actual != 0 else NIVEL_PATRON


def _vuelta(sentido):
    nombre = "lr" if sentido == 1 else "ll"
    poner_luz(nombre, 1)
    direccion(sentido)
    traccion(_nivel_patron())
    esperar(TIEMPO_CIRCULO)
    poner_luz(nombre, 0)


def patron_circle():
    print("Circle")
    _vuelta(1)
    detener_todo()


def patron_infinite():
    print("Infinite")
    _vuelta(1)
    _vuelta(-1)
    detener_todo()


def patron_zigzag():
    print("ZigZag")
    traccion(_nivel_patron())
    sentido = 1
    for _ in range(REPETICIONES_ZIGZAG):
        nombre = "lr" if sentido == 1 else "ll"
        poner_luz(nombre, 1)
        direccion(sentido)
        esperar(TIEMPO_ZIGZAG)
        poner_luz(nombre, 0)
        sentido = -sentido
    detener_todo()


def patron_especial():
    global _en_patron
    _en_patron = True             # mientras dura, el LDR no toca las luces
    try:
        _patron_especial()
    finally:
        _en_patron = False


def _patron_especial():
    print("especial")
    previas = dict(luces)
    ida_vuelta = ("lr", "lf", "lb", "ll", "lb", "lf", "lr")
    traccion(_nivel_patron())
    for _ in range(2):
        for nombre in ida_vuelta:
            for k in luces:
                luces[k] = 0
            luces[nombre] = 1
            actualizar_luces()
            esperar(0.15)
    detener_todo()
    esperar(0.3)
    luces["lb"] = 1
    actualizar_luces()
    traccion(-_nivel_patron())
    esperar(1.5)
    detener_todo()
    for _ in range(3):
        for k in luces:
            luces[k] = 1
        actualizar_luces()
        esperar(0.2)
        for k in luces:
            luces[k] = 0
        actualizar_luces()
        esperar(0.2)
    for k in previas:
        luces[k] = previas[k]
    actualizar_luces()


# ---------------------------------------------------------------
# INTERPRETE DE COMANDOS
# ---------------------------------------------------------------
def procesar(linea):
    partes = linea.strip().split()
    if not partes:
        return
    c = partes[0].lower()

    if c == "luz":
        if len(partes) == 1:
            cmd_luz_lectura()
        elif partes[1] in ("0", "1"):
            cmd_monitor(int(partes[1]))
        else:
            print("Uso: luz | luz 1 (monitor) | luz 0 (detener)")
        return

    if c == "ip":
        print("IP de la Pico:", _ip, "- puerto", PUERTO)
        return

    if c == "ayuda":
        cmd_ayuda()
        return

    if c == "modo":
        cmd_modo(partes[1].lower() if len(partes) > 1 else "")
        return

    if c == "umbral":
        if len(partes) == 1:
            print("umbral -> enciende bajo {:.2f} V, apaga sobre {:.2f} V".format(
                V_OSCURO, V_CLARO))
        else:
            cmd_umbral(partes[1])
        return

    if c == "ldrinv":
        if len(partes) == 2 and partes[1] in ("0", "1"):
            cmd_ldrinv(int(partes[1]))
        else:
            print("Uso: ldrinv 1 (invertido) | ldrinv 0")
        return

    if c == "abc":
        if len(partes) != 2:
            print("Uso: abc 101")
        else:
            cmd_abc(partes[1])
        return

    if c in ("circle", "infinite", "zigzag", "especial", "stop", "test"):
        if c == "circle":
            patron_circle()
        elif c == "infinite":
            patron_infinite()
        elif c == "zigzag":
            patron_zigzag()
        elif c == "especial":
            patron_especial()
        elif c == "test":
            cmd_test()
        else:
            detener_todo()
            print("stop")
        return

    if c in ("pwm", "dir", "lf", "lb", "ll", "lr", "ldr", "dec"):
        if len(partes) != 2:
            print("Uso:", c, "<valor>")
            return
        try:
            v = int(partes[1])
        except ValueError:
            print("Valor invalido:", partes[1])
            return
        if c == "pwm":
            cmd_pwm(v)
        elif c == "dir":
            cmd_dir(v)
        elif c == "ldr":
            cmd_ldr(v)
        elif c == "dec":
            cmd_dec(v)
        else:
            cmd_luz(c, v)
        return

    print("Comando desconocido:", c)


def _tick_ldr(t):
    try:
        if monitor_ldr:
            cmd_luz_lectura()
        revisar_ldr()
        revisar_dec()
    except Exception as e:
        global _ult_error
        if str(e) != _ult_error:          # no repetir el mismo error 3 veces por segundo
            _ult_error = str(e)
            print("Error en LDR/DEC:", e)


# ---------------------------------------------------------------
# SERVIDOR WIFI (TCP) + DESCUBRIMIENTO (UDP)
# ---------------------------------------------------------------
def _cargar_wifi():
    """Usa wifi.json (lo guarda la GUI) o, si no existe, las constantes de arriba."""
    try:
        import json
        with open("wifi.json") as f:
            d = json.load(f)
        return d["ssid"], d["clave"]
    except Exception:
        return WIFI_SSID, WIFI_CLAVE


def _led(v):
    """LED de la Pico W: apagado = sin energia | parpadea = arrancando/buscando WiFi |
    encendido fijo = servidor listo."""
    try:
        Pin("LED", Pin.OUT).value(v)
    except Exception:
        pass


def conectar_wifi():
    global _ip, _wlan
    _led(1)
    import network
    wlan = network.WLAN(network.STA_IF)
    _wlan = wlan
    wlan.active(False)
    time.sleep_ms(500)
    wlan.active(True)
    try:
        wlan.config(pm=0xa11140)       # sin ahorro de energia: conecta mas estable
    except Exception:
        pass
    ssid, clave = _cargar_wifi()
    for intento in range(2):
        if wlan.isconnected():
            break
        _print("Conectando a", ssid, "(intento", intento + 1, ") ...")
        wlan.connect(ssid, clave)
        t0 = time.time()
        led = 0
        while not wlan.isconnected() and time.time() - t0 < 20:
            led ^= 1
            _led(led)                  # parpadeo: el codigo corre y busca la red
            time.sleep_ms(300)
        if not wlan.isconnected():
            wlan.disconnect()
            time.sleep_ms(500)
    if not wlan.isconnected():
        raise RuntimeError("No se pudo conectar al WiFi (estado {})".format(wlan.status()))
    ip = wlan.ifconfig()[0]
    _ip = ip
    _led(1)
    _print("WiFi OK. IP de la Pico:", ip, "- puerto", PUERTO)
    return ip


_ult_wifi = [0]


class _Cli:
    """Un cliente TCP conectado."""

    def __init__(self, sock, ip):
        self.sock = sock
        self.ip = ip
        self.buf = b""
        self.t = time.ticks_ms()   # ultimo dato recibido (para comandos sin salto de linea)
        self.rol = "tcp"           # pasa a 'gui' si la interfaz de la compu se identifica
        self.cmds = 0              # comandos de control que ha ejecutado

    def nombre(self):
        return self.rol + " " + self.ip


# Comandos que cualquier cliente puede mandar aunque no tenga el control
LIBRES = ("modo", "ayuda", "ip", "stop", "luz", "gui")
# Comandos que no se repiten (eco) a los demas clientes
SIN_ECO = ("luz", "ip", "ayuda")


def _hay_gui():
    for c in _clientes:
        if c.rol == "gui":
            return True
    return False


def _tiene_control(c):
    """Quien puede mover el carro.
    - Sin GUI conectada: cualquier cliente TCP (el celular solo).
    - Con GUI en modo PC: solo la GUI. Con GUI en modo celular: todos menos la GUI."""
    if not _hay_gui():
        return True
    if _modo_celular:
        return c.rol != "gui"
    return c.rol == "gui"


def _ejecutar(c, linea):
    """Interpreta una linea recibida de un cliente (bytes)."""
    texto = "".join([chr(b) for b in linea if 32 <= b < 127]).strip()   # ignora basura/telnet
    if not texto:
        return
    cmd = texto.split()[0].lower()

    if cmd == "ping":                      # latido: la GUI comprueba que la Pico sigue viva
        _enviar(c, "pong")
        return

    if cmd == "gui":                       # la GUI de la compu se identifica
        c.rol = "gui"
        _enviar(c, "Rol: GUI")
        _enviar(c, "MODO " + ("celular" if _modo_celular else "pc"))
        _enviar(c, "ldr -> " + ("1" if modo_ldr else "0"))
        _enviar(c, "dec -> " + ("1" if modo_dec else "0"))
        return

    if cmd not in LIBRES:
        if not _tiene_control(c):
            if c.rol == "gui":
                _enviar(c, "ERR: modo celular activo; el control lo tiene el celular")
            else:
                _enviar(c, "ERR: la GUI tiene el control. Envia 'modo celular' para tomarlo")
            return
        c.cmds += 1

    if cmd not in SIN_ECO:
        _difundir("[" + c.nombre() + "] " + texto, c)   # los demas ven quien mando que
    try:
        procesar(texto)
    except Exception as e:
        detener_todo()
        print("Error:", e, "- motores detenidos")
    c.t = time.ticks_ms()          # los patrones bloquean la Pico: no cuentan como silencio


def _recibir(c, datos):
    """Acumula bytes y ejecuta cada linea completa (acepta \\n, \\r o \\r\\n)."""
    c.buf += datos
    c.t = time.ticks_ms()
    c.buf = c.buf.replace(b"\r", b"\n")
    while b"\n" in c.buf:
        linea, c.buf = c.buf.split(b"\n", 1)
        _ejecutar(c, linea)


def _vaciar_inactivos(ahora, poll=None):
    """Muchas apps de TCP client mandan 'pwm 2' SIN salto de linea: si pasan 250 ms
    sin mas datos, se toma lo recibido como una linea completa.
    Ademas vigila a la GUI, que manda un 'ping' cada 2 s: si se queda callada
    (compu dormida, WiFi caido) el carro se detiene y, pasado un rato, se libera su lugar."""
    for c in list(_clientes):
        silencio = time.ticks_diff(ahora, c.t)
        if c.buf and silencio > 250:
            linea = c.buf
            c.buf = b""
            _ejecutar(c, linea)
        elif c.rol == "gui":
            if silencio > 8000 and c.cmds > 0 and _tiene_control(c) and nivel_actual != 0:
                detener_todo()
                print("GUI sin senal: carro detenido")
            if silencio > 25000 and poll is not None:
                _quitar(poll, c)       # GUI muerta: libera el lugar (y el control del celular)


def _cliente_de(sock):
    for c in _clientes:
        if c.sock is sock:
            return c
    return None


def _quitar(poll, c):
    ctl = c.cmds > 0 and _tiene_control(c)   # ¿estaba manejando el carro?
    try:
        poll.unregister(c.sock)
    except Exception:
        pass
    try:
        c.sock.close()
    except Exception:
        pass
    if c in _clientes:
        _clientes.remove(c)
    if ctl:
        detener_todo()                       # el que manejaba se fue: parar por seguridad
    print("Cliente desconectado:", c.ip, "- quedan", len(_clientes))


def _admitir(poll, sock, ip):
    import select
    if len(_clientes) >= MAX_CLIENTES:       # lleno: se saca al mas antiguo que no sea la GUI
        victima = _clientes[0]
        for c in _clientes:
            if c.rol != "gui":
                victima = c
                break
        _quitar(poll, victima)
    sock.settimeout(0.5)
    c = _Cli(sock, ip)
    _clientes.append(c)
    poll.register(sock, select.POLLIN)
    _enviar(c, "Formula E CES Tec - Pico {}:{}  (escribe 'ayuda')".format(_ip, PUERTO))
    print("Cliente conectado:", ip, "- total", len(_clientes))


def servidor():
    import socket
    import select

    conectar_wifi()

    srv = socket.socket()
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind(("0.0.0.0", PUERTO))
    srv.listen(3)

    udp = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    udp.bind(("0.0.0.0", PUERTO_BUSQUEDA))

    try:
        Pin("LED", Pin.OUT).value(1)   # LED de la Pico encendido = servidor listo
    except Exception:
        pass

    poll = select.poll()
    poll.register(srv, select.POLLIN)
    poll.register(udp, select.POLLIN)
    try:
        while True:
            for obj, ev in poll.poll(100):
                if obj is udp:
                    d, a = udp.recvfrom(64)
                    if d.startswith(b"FORMULAE?"):
                        udp.sendto(b"FORMULAE-PICO", a)

                elif obj is srv:
                    nuevo, a = srv.accept()
                    _admitir(poll, nuevo, a[0])

                else:
                    c = _cliente_de(obj)
                    if c is None:
                        continue
                    datos = b""
                    if ev & select.POLLIN:
                        try:
                            datos = c.sock.recv(128)
                        except OSError:
                            datos = b""
                    if not datos:
                        _quitar(poll, c)
                        continue
                    _recibir(c, datos)

            ahora = time.ticks_ms()
            _vaciar_inactivos(ahora, poll)
            if time.ticks_diff(ahora, _ult_wifi[0]) > 2000:      # cada 2 s revisa el WiFi
                _ult_wifi[0] = ahora
                if _wlan is not None and not _wlan.isconnected():
                    raise RuntimeError("se perdio el WiFi")
    finally:
        for c in list(_clientes):
            try:
                c.sock.close()
            except Exception:
                pass
        while _clientes:
            _clientes.pop()
        srv.close()
        udp.close()


def main():
    global _timer
    detener_todo()
    actualizar_luces()
    _timer = Timer(-1, mode=Timer.PERIODIC, period=300, callback=_tick_ldr)
    if USAR_WIFI:
        # Funcionando solo con bateria no hay nadie en la consola: se reintenta SIEMPRE
        # (si se cae el WiFi o falla el servidor, el carro se detiene y se reconecta solo).
        while True:
            try:
                servidor()
            except KeyboardInterrupt:
                raise
            except Exception as e:
                detener_todo()
                try:
                    Pin("LED", Pin.OUT).value(0)   # LED apagado = sin conexion, reintentando
                except Exception:
                    pass
                _print("WiFi/servidor fallo:", e, "- reintento en 3 s")
                time.sleep(3)
    print("Formula E CES Tec listo (consola). Comandos: pwm dir lf lb ll lr ldr "
          "Circle Infinite ZigZag especial stop test luz dec abc")
    while True:
        linea = input("> ")
        try:
            procesar(linea)
        except Exception as e:
            detener_todo()
            print("Error:", e, "- motores detenidos")


try:
    main()
except KeyboardInterrupt:
    detener_todo()
    dec_enviar(0)
    luces["lr"] = luces["lf"] = luces["lb"] = luces["ll"] = 0
    actualizar_luces()
    _print("Detenido")
    #iPhone de Esteban
    #TNsS-inzE-OCr0-UzYL
