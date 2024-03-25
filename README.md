# Soal No 4
# Dikerjakan oleh Raditya Hardian Santoso (5027231033)


Repositori ini berisi skrip untuk memantau penggunaan RAM dan ukuran direktori secara berkala dan mencatat metriknya ke dalam file. Dua skrip utama yang disertakan adalah:

- `minute_log.sh`: Skrip ini mencatat metrik RAM dan ukuran direktori setiap menit.
- `aggregate_minutes_to_hourly_log.sh`: Skrip ini menggabungkan metrik yang dicatat oleh `minute_log.sh` menjadi ringkasan per jam, termasuk nilai minimum, maksimum, dan rata-ratanya.

Dalam soal ini, kita diminta untuk membuat program yang memonitor ram dan size directory /home/{user} dengan cara memasukkan metrics kedalam file log secara periodik.

## minute_log.sh

Isi skrip saya :

```bash
#!/bin/bash

# Navigasi ke direktori log
cd /home/kali/log || exit

# Membuat nama file log berdasarkan timestamp
logfile="metrics_$(date +'%Y%m%d%H%M').log"

# Mendapatkan metrik RAM menggunakan 'free -m'
ram_metrics=$(free -m | awk 'NR==2 {print $2","$3","$4","$5","$6","$7}')

# Mendapatkan ukuran direktori menggunakan 'du -sh'
directory="/home/kali/coba/"
dir_size=$(du -sh $directory | awk '{print $1}')

# Menyimpan metrik RAM, ukuran swap, dan ukuran direktori ke dalam file log
echo "mem_total,mem_used,mem_free,mem_shared,mem_buff,mem_available,swap_total,swap_used,swap_free,dir_path,dir_size" > "$logfile"
echo "$ram_metrics,$(swapon -s | awk 'NR==2 {print $2","$3","$4}'),$directory,$dir_size" >> "$logfile"

# Mengatur izin agar file log hanya dapat dibaca oleh pemiliknya
chmod 600 "$logfile"

```
Script bash yang pertama (minute_log.sh) akan memasukkan metrics ke dalam file dengan format metrics_{YmdHms}.log setiap menitnya. Script ini juga akan membuat folder dalam directory log dengan format metrics_agg_$(YmdH) dengan nilai H merupakan 1 jam sebelum script dijalankan. Tujuannya agar file log yang digenerate tiap menit tidak berceceran. hasil run:

![alt text](https://cdn.discordapp.com/attachments/1176233896292122634/1221504545960755250/image.png?ex=6612d1c2&is=66005cc2&hm=a610467e45968273627aa85c3b476968f1c8b04ce8b2cd0e9b107fd7cf9996f6&)

Karena minute_log.sh harus dijalankan tiap menit, kita dapat menambahkan line berikut pada crontab (crontab -e):
```bash
* * * * * /{path minute_log.sh}
```

Cara melihat hasil : 
```bash
tail -n 10 $(ls -t /home/kali/log/metrics_*.log | head -n 1)
```

## aggregate_minutes_to_hourly_log.sh

Skrip ini menggabungkan file log per menit menjadi ringkasan per jam, menghitung nilai minimum, maksimum, dan rata-rata untuk setiap metrik.

Script bash yang kedua (aggregate_minutes_to_hourly_log.sh) akan menggabungkan file log yang dibuat minute_log.sh dan akan menyimpan nilai minimum, maximum, dan rata-rata tiap metrics setiap jam. Caranya adalah dengan membaca seluruh file log yang berada dalam folder metrics_agg_$(YmdH).

Isi skrip saya :

```bash
#!/bin/bash

# Navigasi ke direktori log
cd /home/kali/log || exit

# Membuat nama file log agregasi berdasarkan timestamp
agglogfile="metrics_agg_$(date +'%Y%m%d%H').log"

# Mendapatkan kolom yang akan diambil rata-rata, minimum, dan maksimum
columns="2,3,4,5,6,7,8,9,10,11,12"

# Mencatat nilai minimum, maksimum, dan rata-rata
minimum=$(awk -F',' 'NR>1 {print $'$columns'}' metrics_*.log | sort -t, -n | head -n 1)
maximum=$(awk -F',' 'NR>1 {print $'$columns'}' metrics_*.log | sort -t, -n | tail -n 1)
average=$(awk -F',' -v col="$columns" 'BEGIN {sum=0; count=0} NR>1 {split(col, arr, ","); for (i in arr) sum+=$arr[i]; count++} END {print sum/count}' metrics_*.log)

# Mengubah satuan hasil
minimum=$(echo $minimum | sed 's/M/ MB/g')
maximum=$(echo $maximum | sed 's/M/ MB/g')
average=$(printf "%.2f" $average) # Mengatur dua angka di belakang koma untuk rata-rata

# Menyimpan hasil agregasi ke dalam file log
echo "type,mem_total,mem_used,mem_free,mem_shared,mem_buff,mem_available,swap_total,swap_used,swap_free,path,path_size" > "$agglogfile"
echo "minimum,$minimum" >> "$agglogfile"
echo "maximum,$maximum" >> "$agglogfile"
echo "average,$average" >> "$agglogfile"

# Mengatur izin agar file log hanya dapat dibaca oleh pemiliknya
chmod 600 "$agglogfile"

```

Untuk mendapatkan nilai average, kita dapat mencatat index, yaitu jumlah file log yang terdapat dalam folder. Selanjutnya kita juga akan mencatat nilai sum/total dari masing-masing metrics yang kemudian akan dibagi dengan index untuk mendapatkan nilai average.

Selanjutnya, untuk mendapat nilai minimum dan maximum kita dapat menyimpan masing-masing value metrics ke dalam array lalu melakukan sort sehingga nilai minimum masing-masing metrics akan berada pada index paling awal dan nilai maximum masing-masing metrics akan berada pada index paling akhir. Kita juga akan menggunakan chown agar log hanya dapat dibaca user.

hasil run:

![alt text](https://cdn.discordapp.com/attachments/1176233896292122634/1221504764857028608/image.png?ex=6612d1f6&is=66005cf6&hm=ca3e8774192fa09367ce9ed864cf1756ccffc99fc2c3adb462e53d329592f91b&)


Agar aggregate_minutes_to_hourly_log.sh dapat dijalankan tiap jam, kita dapat menambahkan line berikut pada crontab:

```bash
0 * * * * /{path aggregate_minutes_to_hourly_log.sh}
```

Cara melihat hasil : 
```bash
tail -n 10 $(ls -t /home/kali/log/metrics_agg*.log | head -n 1)
```
