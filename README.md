# O'zgaruvchilar (svetofor rangi va mashina tezligi)
svetofor_rangi = "qizil"  # "qizil", "sariq", "yashil" yoki boshqa rang
tezlik = 0  # Mashina tezligi

# Shartlarni tekshirish
if svetofor_rangi == "qizil":
    if tezlik == 0:
        print("To'g'ri, to'xtab turibsiz.")
    else:
        print("Taqiqlangan! Sizga jarima yozildi!")

elif svetofor_rangi == "sariq":
    if tezlik > 50:
        print("Tezlikni kamaytiring va to'xtashga tayyorlaning!")
    else:
        print("To'xtab kuting.")

elif svetofor_rangi == "yashil":
    print("Yo'lingiz ochiq, xavfsiz harakatlaning!")

else:
    print("Svetofor buzilgan, tartibga soluvchiga qarang!")
    
