---
layout: post
title:  "Solusi Sistem Persamaan Linear dengan Inverse Matriks"
date:   2020-11-16 23:42:54 +0700
categories: jekyll update
---

## Intro
Seminggu yang lalu saya iseng melihat-lihat kembali repo-repo di github saya dan menemukan project semasa kuliah yang menarik untuk di-rewrite. Secara ringkas, project ini bertujuan menyelesaikan sistem persamaan linear menggunakan metode invers matriks. Misal diberikan `n` buah persamaan linear dengan `n` variabel berikut:

```
a1.x1 + b1.x2 + ... + n1.xn = y1
a2.x1 + b1.x2 + ... + n2.xn = y2
      .
      .
an.x1 + bn.x1 + ... + nn.xn = yn
```

Di sini kita ingin mencari nilai dari `[x1, x2, ..., xn]` yang memenuhi sistem persamaan tersebut. Secara umum terdapat tiga jenis solusi, yaitu: tepat satu solusi, tidak ada solusi, dan terdapat tak-berhingga solusi. Pada post ini, akan dibatasi untuk membahas kasus pertama saja.

## A Little Bit of Math
Sistem persamaan yang sebelumnya sebenarnya bisa diubah ke bentuk perkalian matriks `AX = Y` seperti berikut:

```
a1 b1 .. n1     x1     y1
a2 b2 .. n2     x2     y2
 .  .     .  *   .  =   .
 .  .     .      .      .
an bn .. nn     xn     yn
-------------------------
      A         X   =   Y
```

Persamaan `AX = Y` ini ekuivalen dengan `X = inverse(A)*Y`. Memperkenalkan matriks identitas `I` ke ekuivalensi tersebut membuatnya menjadi lebih menarik.

```
AX = IY  <=>  IX = inverse(A)*Y
```

Dari ekuivalensi terakhir dapat dilihat bahwa jika kita dapat melakukan operasi baris untuk mengubah matriks `A` di kiri menjadi `I`, secara otomatis kita akan mengubah matriks `I` yang di kanan menjadi `inverse(A)`.

## Struktur Data Matriks
Project ini saya rewrite ke bahasa Kotlin karena syntaxnya yang ringkas dan elegan. Di bahasa Kotlin sendiri belum ada struktur data khusus untuk menangani persoalan terkait matriks. Oleh sebab itu perlu diimplementasikan struktur data khusus. Dalam implementasinya, array bisa digunakan untuk menyimpan elemen-elemen matriks, namun perlu diatur `getter` dan `setter` agar sesuai dengan indeks yang dituju.

```kotlin
class Matrix<T : Number>(val rows: Int, val cols: Int) {

  lateinit var elements: Array<T>

  private fun index(r: Int, c: Int) = r * cols + c

  operator fun get(r: Int, c: Int) = elements[index(r, c)]

  operator fun set(r: Int, c: Int, element: T) {
    elements[index(r, c)] = element
  }

  constructor(rows: Int, cols: Int, elements: Array<T>) : this(rows, cols) {
    this.elements = elements
    for (r in 0 until rows) {
      for (c in 0 until cols) set(r, c, elements[index(r, c)])
    }
  }
}
```

Pada struktur data matriks ini, disimpan data berupa ukuran baris (`rows`), ukuran kolom (`cols`), dan array elemen-elemen matriks (`elements`). Di sini juga dibutuhkan fungsi `index(row, col)` untuk memetakan indeks matriks ke indeks array yang digunakan. Selain itu juga didefinisikan sebuah constructor sehingga objek matriks dapat dibuat dengan cara seperti berikut:

```kotlin
/*
  1, 2, 3
  4, 5, 6
*/
Matrix<Int> matrix = Matrix(2, 3, arrayOf(1,2,3,4,5,6))
```
## Determinan (Rules of Sarrus)
Salah satu syarat agar matriks memiliki inverse adalah dengan mengecek apakah nilai determinan dari matriks tersebut tidak nol. [Rules of Sarrus](https://en.wikipedia.org/wiki/Rule_of_Sarrus) dapat digunakan untuk menghitung nilai determinan dari suatu matriks.

![src: wikipedia](https://upload.wikimedia.org/wikipedia/commons/thumb/2/2d/Schema_sarrus-regel.png/440px-Schema_sarrus-regel.png)

Dari metode Sarrus ini, kita perlu melakukan iterasi secara horizontal (berdasarkan kolom matriks). Untuk satu iterasi di kolom tersebut, kita menghitung hasil kali dari elemen-elemen secara diagonal dari atas (merah) dan diagonal dari bawah (biru). Elemen diagonal dari atas pada iterasi `[c,r]` berada pada baris `r`, sedangkan untuk elemen diagonal dari bawah berada pada baris `rows-r-1`. Sedangkan untuk indeks kolom dari kedua diagonal ini sama yaitu `(r+c)%cols`. Operasi modulo digunakan agar indeks kolom tidak mengakses area di luar matriks (abu-abu). Metode ini berlaku untuk matriks berdimensi 3x3 ke atas. Untuk dimensi di bawah itu perlu ditangani secara manual.

```kotlin
fun <T : Number> det(matrix: Matrix<T>): T {
  require(matrix.rows == matrix.cols)

  if (matrix.rows == 1) return matrix[0,0]
  if (matrix.rows == 2) return (
    matrix[0,0] * matrix[1,1] - 
    matrix[0,1] * matrix[1,0]
  ) as T

  var det = 0.0
  for (c in 0 until matrix.cols) {
    var incValue = 1.0
    var decValue = 1.0

    for (r in 0 until matrix.rows) {
      incValue *= matrix[r, (c+r)%matrix.cols].toDouble()
      decValue *= matrix[matrix.rows-r-1, (c+r)%matrix.cols].toDouble()
    }
    det += (incValue - decValue)
  }
  return det as T
}
```

## Inverse Matriks (Gaussian Elimination)

Dari ekuivalensi `AX = IY  <=> IX = inverse(A)*Y` dapat digunakan operasi baris [Gaussian Elimination](https://en.wikipedia.org/wiki/Gaussian_elimination#:~:text=Gaussian%20elimination%2C%20also%20known%20as,the%20corresponding%20matrix%20of%20coefficients.&text=The%20method%20is%20named%20after,Gauss%20(1777%E2%80%931855).) untuk mengubah matriks `A` menjadi `I` (sisi kiri) dan `I` menjadi `inverse(A)` (sisi kanan). Adapun strategi yang dapat digunakan untuk melakukan operasi baris tersebut yaitu:
- Pada baris `R` perlu ditemukan focus-diagonal (`fd`) yaitu elemen pertama bukan nol di baris tersebut.
- Elemen-elemen pada baris `R` dikalikan dengan nilai `1/fd`.
- Untuk baris lainnya diambil elemen yang satu kolom dengan `fd`, misalkan `k`. Baris tersebut dikurangi dengan `k*R`.

```kotlin
fun <T : Number> inverse(matrix: Matrix<T>): Matrix<Double>? {
  require(matrix.rows == matrix.cols)

  // check matrix invertible
  if (det(matrix).equalsDelta(0.0)) return null

  // copy matrix
  val inverse = Matrix.diagonal(matrix.rows, 1.0, 0.0)
  val temp: Matrix<Double> = Matrix(
    matrix.rows, 
    matrix.cols, 
    matrix.elements.map { it.toDouble() }.toTypedArray()
  )
  for (r in 0 until matrix.rows) {
    for (c in 0 until matrix.cols) temp[r,c] = matrix[r,c].toDouble()
  }

  for (fdRow in 0 until matrix.rows) {
    // find focus-diagonal
    while (fdCol < matrix.cols 
      && temp[fdRow, fdCol].equalsDelta(0.0)) fdCol++

    // matrix not invertible if all row is zero
    if (fdCol == matrix.cols) return null

    // multiply with 1/fd
    val scalingFactor = 1 / temp[fdRow, fdCol]
    for (c in 0 until matrix.cols) {
      temp[fdRow, c] *= scalingFactor
      inverse[fdRow, c] *= scalingFactor
    }

    // subtract with k*R
    for (r in 0 until matrix.rows) {
      if (r == fdRow) continue
      val subFactor = temp[r, fdCol]
      for (c in 0 until matrix.cols) {
        temp[r, c] -= subFactor * temp[fdRow, c]
        inverse[r, c] -= subFactor * inverse[fdRow, c]
      }
    }
  }

  return inverse
}
```

## Solusi Sistem Persamaan Linear
Setelah mendapatkan inverse dari matriks koefisien sistem persamaan (`inverse(A)`), solusi dari sistem persamaan mudah dihitung menggunakan cross-product dari matriks inverse dan matriks konstantanya. Di sini saya menggunakan perhitungan yang paling umum untuk cross-product dua buah matriks seperti berikut:

```kotlin
// matrix cross-product
infix fun <R : Number> x(other: Matrix<R>): Matrix<Double> {
  require(cols == other.rows)
  val res = Matrix(rows, other.cols, Array(rows * other.cols) { 0.0 })
  for (i in 0 until rows) {
    for (j in 0 until other.cols) {
      for (k in 0 until cols) res[i,j] += (this[i,k] * other[k,j]).toDouble()
    }
  }
  return res
}
```

Dengan fungsi inverse dan cross-product di atas, solusi dari sistem persamaan yang ingin diketahui dapat dihitung seperti berikut:

```kotlin
// lhs (left-hand-side), rhs (right-hand-side)
fun <T : Number> solve(lhs: Matrix<T>, rhs: Matrix<T>): Matrix<Double>?{
  return inverse(lhs)?.let { it x rhs }
}
```

## Show Me the Code
Lebih lanjut tentang post ini, silahkan kunjungi repository github-nya [di sini](https://github.com/dionsaputra/matrix-equation-solver).