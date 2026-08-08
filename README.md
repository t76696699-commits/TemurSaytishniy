# ─── Generatorlar bilan ishlash ──────────────────────────────────────────
import sys
from itertools import islice
import time

# 1) Eng oddiy yield
def son_qatori(n):
    for i in range(n):
        yield i * 2

g = son_qatori(5)
print(type(g))
print(list(g))                 # [0, 2, 4, 6, 8]

# 2) Generator vs list — xotira solishtirish
katta_list = [x for x in range(10**6)]
katta_gen  = (x for x in range(10**6))
print("list  bayt:", sys.getsizeof(katta_list))
print("gen   bayt:", sys.getsizeof(katta_gen))

# 3) Cheksiz fibonacchi
def fib():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

# Birinchi 10 tasini olamiz — generatorning kuchi
print(list(islice(fib(), 10)))

# 4) Pipeline pattern (logni filterlash)
def manba():
    yield "INFO: server start"
    yield "ERROR: db timeout"
    yield "INFO: request 200"
    yield "ERROR: auth failed"
    yield "INFO: shutdown"

def faqat_error(qatorlar):
    for q in qatorlar:
        if q.startswith("ERROR"):
            yield q

def vaqt_qoshish(qatorlar):
    for q in qatorlar:
        yield f"[{time.strftime('%H:%M:%S')}] {q}"

oqim = vaqt_qoshish(faqat_error(manba()))
for q in oqim:
    print(q)

# 5) Generator faqat bir marta
g = (x for x in range(3))
print("birinchi:", list(g))    # [0, 1, 2]
print("ikkinchi:", list(g))    # []  — bo'sh

# 6) yield from — birlashtirish
def birlashma(*iterables):
    for it in iterables:
        yield from it

print(list(birlashma([1, 2], (3, 4), range(5, 8))))
# [1, 2, 3, 4, 5, 6, 7]
