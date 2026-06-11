# twoKeyHeap
import sys
import re
import unicodedata

#Turkce karakterleri ASCII'ye donustur

TR_MAP = str.maketrans("çÇğĞıİöÖşŞüÜ", "cCgGiIoOsOuU")
# Duzeltme: ş->s, Ş->S
TR_MAP = str.maketrans(
    "çÇğĞıİöÖşŞüÜ",
    "cCgGiIoOsSuU"
)

def normalize(word: str) -> str:
    """Kelimeyi kucuk harfe cevir, Turkce karakterleri ASCII yap."""
    word = word.translate(TR_MAP)
    # Kalan unicode karakterleri ASCII'ye donustur (aksan vb.)
    word = unicodedata.normalize("NFD", word)
    word = "".join(c for c in word if unicodedata.category(c) != "Mn")
    return word.lower()

# HEAP DUGUMU

class HeapNode:
    """
    Heap dugumu:
      - letter  : ilk harf (ASCII, kucuk)     -> 1. anahtar
      - count   : kac kez gectigi             -> 2. anahtar (max)
      - word    : orijinal (normalize edilmis) kelime
    """
    def __init__(self, word: str, letter: str, count: int = 1):
        self.word   = word
        self.letter = letter      # 1. anahtar
        self.count  = count       # 2. anahtar

    def __repr__(self):
        return f"({self.letter}|{self.word}:{self.count})"

# IKI ANAHTARLI MAX-HEAP
# (1. anahtar: letter ASCII artan, 2. anahtar: count azalan)
# Python heapq min-heap oldugu icin: kucuk = once gelmeli
# Oncelik: (letter_ascii ARTAN, -count ARTAN yani count AZALAN)

import heapq

class TwoKeyHeap:
    """
    2 anahtarli heap:
      Ana yapi: list of [priority_key, HeapNode]
      priority_key = (ord(letter), -count)   -> min-heap ile max-count saglanir

    Kelime guncellemesi icin lookup dict tutulur: normalized_word -> HeapNode
    Heap'te guncelleme icin "lazy deletion" yontemi kullanilir.
    """

    def __init__(self):
        self._heap   = []          # [(ord(l), -count, word), ...]
        self._table  = {}          # word -> count  (gercek guncel deger)
        self._removed = set()      # artik gecersiz girişler icin

    def _push(self, word: str, letter: str, count: int):
        entry = (ord(letter), -count, word)
        heapq.heappush(self._heap, entry)

    def insert_or_update(self, word: str):
        """
        Kelimeyi heap'e ekle veya mevcutsa count'u artir.
        Guncelleme: lazy deletion - eski giris gecersiz sayilir,
        yeni giris push edilir.
        """
        letter = word[0]  # normalize edilmis kelimenin ilk harfi

        if word in self._table:
            # Mevcut -> count artir, yeni entry ekle (eski lazy-delete ile iptal)
            self._removed.add((ord(letter), -self._table[word], word))
            self._table[word] += 1
        else:
            self._table[word] = 1

        count = self._table[word]
        self._push(word, letter, count)

    def _clean_top(self):
        """Gecersiz (removed) girisleri heap'in basından temizle."""
        while self._heap and self._heap[0] in self._removed:
            removed_entry = heapq.heappop(self._heap)
            self._removed.discard(removed_entry)

    def peek(self):
        self._clean_top()
        if self._heap:
            ord_l, neg_count, word = self._heap[0]
            return word, chr(ord_l), -neg_count
        return None

    def pop(self):
        """En oncelikli dugumu cikart (A harfi, en yuksek count)."""
        self._clean_top()
        while self._heap:
            ord_l, neg_count, word = heapq.heappop(self._heap)
            entry = (ord_l, neg_count, word)
            if entry in self._removed:
                self._removed.discard(entry)
                continue
            # Gercekten var mi?
            if word in self._table and self._table[word] == -neg_count:
                del self._table[word]
                return word, chr(ord_l), -neg_count
        return None

    def is_empty(self) -> bool:
        self._clean_top()
        return len(self._heap) == 0

    def size(self) -> int:
        return len(self._table)

    def snapshot(self) -> list:
        """
        Mevcut heap icerigini sirali liste olarak dondur
        (heap'i bozmadan kopya uzerinden).
        """
        result = []
        for word, count in self._table.items():
            letter = word[0]
            result.append((word, letter, count))
        # Sirala: letter ASCII artan, count azalan
        result.sort(key=lambda x: (ord(x[1]), -x[2]))
        return result

# DOSYA OKUMA

def read_words_from_file(filepath: str):
    """
    TXT dosyasini satir satir oku, her satirdan kelimeleri ayikla.
    Sadece alfabetik karakterler alinir, noktalama/sayi atlanir.
    Normalize edilmis (ASCII lowercase) kelime doner.
    Doner: (satir_no, satir_metni, [normalize_kelimeler])
    """
    try:
        with open(filepath, "r", encoding="utf-8", errors="replace") as f:
            lines = f.readlines()
    except FileNotFoundError:
        print(f"HATA: '{filepath}' dosyasi bulunamadi.")
        sys.exit(1)

    result = []
    for i, raw_line in enumerate(lines, start=1):
        # Satirdan kelimeleri ayikla (sadece harf icerenleri al)
        tokens = re.findall(r"[a-zA-ZçÇğĞıİöÖşŞüÜ]+", raw_line)
        norm_words = [normalize(t) for t in tokens if normalize(t)]
        result.append((i, raw_line.rstrip("\n"), norm_words))
    return result

# SATIR SATIR ADIM CIKTISI

def print_step(step_no: int, word: str, heap: TwoKeyHeap):
    snap = heap.snapshot()
    print(f"\n{'='*60}")
    print(f"  Adim {step_no:>4} | Islenen kelime: '{word}'")
    print(f"  Heap boyutu: {heap.size()} benzersiz kelime")
    print(f"  Heap icerigi (letter|kelime:sayi):")
    for w, l, c in snap:
        print(f"    [{l.upper()}]  {w:<20} -> {c}")
    print(f"{'='*60}")

# SONUC YAZDIR

def print_final_results(heap: TwoKeyHeap):
    snap = heap.snapshot()

    print("\n" + "╔" + "═"*58 + "╗")
    print("║" + "   FINAL SIRALAMA SONUCLARI (A'dan Z'ye, Cok->Az)   ".center(58) + "║")
    print("╚" + "═"*58 + "╝")

    # Harfe gore grupla
    from itertools import groupby
    for letter, group in groupby(snap, key=lambda x: x[1]):
        items = list(group)
        print(f"\n  ── Harf: {letter.upper()} ──")
        for rank, (w, l, c) in enumerate(items, start=1):
            bar = "█" * min(c, 30)
            print(f"    {rank:>3}. {w:<20} | {c:>4} kez  {bar}")

    # Toplam siralama (tum kelimeler, count azalan)
    all_sorted = sorted(snap, key=lambda x: -x[2])
    print("\n" + "╔" + "═"*58 + "╗")
    print("║" + "   TOPLAM KELIME SIRALAMASI (En Cok -> En Az)   ".center(58) + "║")
    print("╚" + "═"*58 + "╝")
    for rank, (w, l, c) in enumerate(all_sorted, start=1):
        bar = "█" * min(c, 30)
        print(f"  {rank:>4}. [{l.upper()}] {w:<20} | {c:>4} kez  {bar}")

    total = sum(x[2] for x in snap)
    print(f"\n  Toplam islenen kelime (tekrarli): {total}")
    print(f"  Benzersiz kelime sayisi        : {len(snap)}")

# ANA PROGRAM

def main():
    if len(sys.argv) < 2:
        # PyCharm veya terminal'de argumansiz calistirilirsa kullanicidan al
        print("=" * 60)
        print("  TXT Kelime Sayici - 2 Anahtarli Heap Siralayici")
        print("=" * 60)
        filepath = input("\nTXT dosyasinin yolunu girin (ornek: ornek.txt): ").strip()
        if not filepath:
            print("HATA: Dosya yolu bos birakilamaz.")
            sys.exit(1)
        adim = input("Adim adim heap gosterimi isteniyor mu? (e/h): ").strip().lower()
        show_steps = adim == "e"
    else:
        filepath = sys.argv[1]
        show_steps = "--steps" in sys.argv   # Adim adim gosterim icin --steps flagi

    print(f"\nDosya okunuyor: {filepath}")
    lines = read_words_from_file(filepath)

    heap = TwoKeyHeap()
    step_no = 0

    print(f"\nToplam {len(lines)} satir bulundu.\n")
    print("─" * 60)
    print(" SATIR SATIR ISLEME BASLANIYOR")
    print("─" * 60)

    for line_no, line_text, words in lines:
        if not words:
            print(f"\n  Satir {line_no:>4}: [bos/kelimesiz satir] -> '{line_text[:60]}'")
            continue

        print(f"\n  Satir {line_no:>4}: '{line_text[:60]}'")
        print(f"           Kelimeler: {words}")

        for word in words:
            step_no += 1
            heap.insert_or_update(word)

            if show_steps:
                print_step(step_no, word, heap)
            else:
                letter = word[0]
                count  = heap._table[word]
                print(f"    -> '{word}' eklendi/guncellendi: [{letter.upper()}] {count} kez | Heap boyutu: {heap.size()}")

    # Final sonuclari
    print_final_results(heap)
bu kodun satır satır bölüm bölüm de olır ne yaptığını oranın ne işe yaradğını açıklar mısın mala anlatır gibi 