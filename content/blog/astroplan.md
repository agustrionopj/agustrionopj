---
title: "Penentuan posisi obyek langit dengan Astroplan"
date: 2019-05-01T12:10:22+07:00
draft: false
tags: [astronomi, python]
---

Dalam astronomi, perencanaan pengamatan adalah proses yang penting untuk mengetahui kemungkinan apakah benda langit yang akan kita amati benar-benar bisa diamati dari posisi kita dan pada rentang waktu yang kita rencanakan. *Tool* yang digunakan untuk merencanakan pengamatan sangatlah beragam, misalnya dengan program [Astroplanner](http://www.astroplanner.net/) ataupun dengan program peta langit seperti [Stellarium](https://stellarium.org/) atau sejenisnya. Pada kesempatan ini, saya akan paparkan cara alternatif, yaitu menggunakan modul dalam `python` yaitu `astroplan`. 

Pertama, kita panggil dulu modul-modul pendukung yang digunakan.

```python
%matplotlib inline
%config InlineBackend.figure_format = 'retina'
```


```python
import numpy as np
import matplotlib.pyplot as plt

import seaborn as sns; sns.set(font_scale=1.5)

plt.rcParams['figure.figsize'] = (15, 10) # ukuran gambar diperbesar
plt.rcParams['font.size'] = 16
```


```python
import astropy.units as u
from astropy.time import Time
from astropy.coordinates import SkyCoord, EarthLocation, AltAz
```

Misalnya, kita akan mengamati Nebula Orion (M42). Maka, kita ambil koordinatnya dari SIMBAD. Proses ini memerlukan koneksi internet. Kita bisa memasukkan koordinatnya secara manual dengan menghilangkan `.from_name` dari syntax dan langsung memasukkan koordinatnya (misal: `obj = SkyCoord('7h20m', '-10d30m20s', frame = 'icrs')`).


```python
obj = SkyCoord.from_name('M42')
```

Lokasi pengamat bisa ditetapkan dengan menggunakan object `EarthLocation` dengan catatan jika lokasi pengamat tidak ada dalam daftar lokasi pengamatan (misalnya kita mengamati dari sebuah tempat yang bukan merupakan situs observatorium resmi).

Anggap lokasi pengamat adalah Observatorium Bosscha, Lembang, Bandung Barat. Waktu pengamatan adalah tanggal 2 Mei 2019 pukul 02:00 WIB. Jika lokasinya adalah observatorium besar yang sudah dikenal luas, maka bisa menggunakan `astropy.coordinates.EarthLocation.get_site_names`


```python
bosscha = EarthLocation(lat=-6.8333*u.deg, lon=107.6167*u.deg, height=1310*u.m) #lokasi Bosscha
utcoffset = 7*u.hour  # Waktu Indonesia Barat
time = Time('2019-05-02 02:00:00') - utcoffset # waktu dalam UT

```

Konversi posisi M42, dari sistem koordinat Equatorial ke sistem koordinat Horizon.


```python
obj_altaz = obj.transform_to(AltAz(obstime=time,location=bosscha))
print("Ketinggian M42 = {0.alt:.2}".format(obj_altaz))
```

    Ketinggian M42 = -7.3e+01 deg
    

Sekarang cari ketinggian M42 ini untuk rentang waktu pengamatan yang diinginkan, misalnya dari pukul 19:00 WIB s.d. 04:00 WIB


```python
midnight = Time('2019-05-02 00:00:00') - utcoffset
delta_midnight = np.linspace(-5, 4, 100)*u.hour # rentang waktu sebelum sampai sesudah tengah malam (5 jam seblm, 4 jam setelah)
frame_at_date = AltAz(obstime=midnight+delta_midnight,
                          location=bosscha) # rentang waktu pengamatan malam hari
obj_altazs_at_date = obj.transform_to(frame_at_date) # alt,az obyek sepanjang rentang waktu pengamatan
```

Ubah alt dan azimuth obyek ke *airmass* dengan `~astropy.coordinates.AltAz.secz`


```python
obj_airmasss_at_date = obj_altazs_at_date.secz
```

Plot airmass sebagai fungsi dari waktu


```python
plt.plot(delta_midnight, obj_airmasss_at_date)
plt.xlim(-5, 4)
plt.ylim(1, 4)
plt.xlabel('Jam dari tengah malam (WIB)')
plt.ylabel('Airmass [$\sec (z)$]')
plt.show()
```


![png](/img/Belajar-Astroplan_15_0.png)

*Airmass* merupakan tebalnya kolom udara yang dilalui cahaya obyek langit, sebelum sampai ke pengamat. Nilai *airmass* terkecil adalah saat obyek tepat berada di meridian pengamat---di sini nilai *airmass* diberi nilai 1. Terlihat pada gambar di atas, nilai *airmass* membesar semenjak Matahari terbenam. Artinya, obyek semakin rendah, mendekati horizon, atau hampir terbenam pada jam-jam tersebut.

Cari posisi Matahari dari tengah hari tanggal 1 Mei 2019 s.d. tengah hari tanggal 2 Mei 2019 dengan menggunakan `~astropy.coordinates.get_sun`.


```python
from astropy.coordinates import get_sun

delta_midnight = np.linspace(-12, 12, 1000)*u.hour # rentang waktu dibagi menjadi 1000 bagian yang sama
times_today_tomorrow = midnight + delta_midnight
frame_today_tomorrow = AltAz(obstime=times_today_tomorrow, location=bosscha)
sun_altazs_today_tomorrow = get_sun(times_today_tomorrow).transform_to(frame_today_tomorrow)

```

Cari posisi Bulan dengan menggunakan `~astropy.coordinates.get_moon`.


```python
from astropy.coordinates import get_moon
moon_today_tomorrow = get_moon(times_today_tomorrow)
moon_altazs_today_tomorrow = moon_today_tomorrow.transform_to(frame_today_tomorrow)

```

Cari posisi (alt,az) dari M42 untuk rentang waktu yang sama.


```python
obj_altazs_today_tomorrow = obj.transform_to(frame_today_tomorrow)

```

Buat plot yang indah.


```python

plt.plot(delta_midnight, sun_altazs_today_tomorrow.alt, color='r', label='sun')
plt.plot(delta_midnight, moon_altazs_today_tomorrow.alt, color=[0.75]*3, ls='--', label='moon')
plt.scatter(delta_midnight, obj_altazs_today_tomorrow.alt,
            c=obj_altazs_today_tomorrow.az, label='obj', lw=0, s=8,
            cmap='viridis')
plt.fill_between(delta_midnight.to('hr').value, 0, 90,
                 sun_altazs_today_tomorrow.alt < -0*u.deg, color='0.5', zorder=0)
plt.fill_between(delta_midnight.to('hr').value, 0, 90,
                 sun_altazs_today_tomorrow.alt < -18*u.deg, color='k', zorder=0)
plt.colorbar().set_label('Azimuth [$^\circ$]')
plt.legend(loc='upper left')
plt.xlim(-12, 12)
plt.xticks(np.arange(13)*2 -12)
plt.ylim(0, 90)
plt.xlabel('Jam dari tengah malam (WIB)')
plt.ylabel('ketinggian [$^\circ$]')
plt.show()
```


![png](/img/Belajar-Astroplan_23_0.png)

Terlihat bahwa saat Matahari terbenam, M42 berada pada ketinggian sekitar $\sim 48^\circ$ dan terus menurun. Obyek ini terbenam sekitar 3 jam sebelum tengah malam.

**[Bosscha Observatory, 12:49 WIB]**