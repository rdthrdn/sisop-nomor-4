# sisop-nomor-4

Dikerjakan oleh Raditya Hardian Santoso (5027231033)

Dalam soal ini, kita diminta untuk membuat program yang memonitor ram dan size directory /home/{user} dengan cara memasukkan metrics kedalam file log secara periodik.
minute_log.sh

#!/bin/bash

Navigasi ke direktori log
cd /home/kali/log || exit

Membuat nama file log berdasarkan timestamp
logfile="metrics_$(date +'%Y%m%d%H%M').log"

Mendapatkan metrik RAM menggunakan 'free -m'
ram_metrics=$(free -m | awk 'NR==2 {print $2","$3","$4","$5","$6","$7}')

Mendapatkan ukuran direktori menggunakan 'du -sh'
directory="/home/kali/coba/"
dir_size=$(du -sh $directory | awk '{print $1}')

Menyimpan metrik RAM, ukuran swap, dan ukuran direktori ke dalam file log
echo "mem_total,mem_used,mem_free,mem_shared,mem_buff,mem_available,swap_total,swap_used,swap_free,dir_path,dir_size" > "$logfile"
echo "$ram_metrics,$(swapon -s | awk 'NR==2 {print $2","$3","$4}'),$directory,$dir_size" >> "$logfile"

Mengatur izin agar file log hanya dapat dibaca oleh pemiliknya
chmod 600 "$logfile"
Script bash yang pertama (minute_log.sh) akan memasukkan metrics ke dalam file dengan format metrics_{YmdHms}.log setiap menitnya. Script ini juga akan membuat folder dalam directory log dengan format metrics_agg_$(YmdH) dengan nilai H merupakan 1 jam sebelum script dijalankan. Tujuannya agar file log yang digenerate tiap menit tidak berceceran. hasil run:

Karena minute_log.sh harus dijalankan tiap menit, kita dapat menambahkan line berikut pada crontab (crontab -e):
* * * * * /{path minute_log.sh}

aggregate_minutes_to_hourly_log.sh
Script bash yang kedua (aggregate_minutes_to_hourly_log.sh) akan menggabungkan file log yang dibuat minute_log.sh dan akan menyimpan nilai minimum, maximum, dan rata-rata tiap metrics setiap jam. Caranya adalah dengan membaca seluruh file log yang berada dalam folder metrics_agg_$(YmdH).
#!/bin/bash

Navigasi ke direktori log
cd /home/kali/log || exit

Membuat nama file log agregasi berdasarkan timestamp
agglogfile="metrics_agg_$(date +'%Y%m%d%H').log"

Mendapatkan kolom yang akan diambil rata-rata, minimum, dan maksimum
columns="2,3,4,5,6,7,8,9,10,11,12"

Mencatat nilai minimum, maksimum, dan rata-rata
minimum=$(awk -F',' 'NR>1 {print $'$columns'}' metrics_*.log | sort -t, -n | head -n 1)
maximum=$(awk -F',' 'NR>1 {print $'$columns'}' metrics_*.log | sort -t, -n | tail -n 1)
average=$(awk -F',' -v col="$columns" 'BEGIN {sum=0; count=0} NR>1 {split(col, arr, ","); for (i in arr) sum+=$arr[i]; count++} END {print sum/count}' metrics_*.log)

Mengubah satuan hasil
minimum=$(echo $minimum | sed 's/M/ MB/g')
maximum=$(echo $maximum | sed 's/M/ MB/g')
average=$(printf "%.2f" $average) # Mengatur dua angka di belakang koma untuk rata-rata

Menyimpan hasil agregasi ke dalam file log
echo "type,mem_total,mem_used,mem_free,mem_shared,mem_buff,mem_available,swap_total,swap_used,swap_free,path,path_size" > "$agglogfile"
echo "minimum,$minimum" >> "$agglogfile"
echo "maximum,$maximum" >> "$agglogfile"
echo "average,$average" >> "$agglogfile"

Mengatur izin agar file log hanya dapat dibaca oleh pemiliknya
chmod 600 "$agglogfile"


Untuk mendapatkan nilai average, kita dapat mencatat index, yaitu jumlah file log yang terdapat dalam folder. Selanjutnya kita juga akan mencatat nilai sum/total dari masing-masing metrics yang kemudian akan dibagi dengan index untuk mendapatkan nilai average.
Selanjutnya, untuk mendapat nilai minimum dan maximum kita dapat menyimpan masing-masing value metrics ke dalam array lalu melakukan sort sehingga nilai minimum masing-masing metrics akan berada pada index paling awal dan nilai maximum masing-masing metrics akan berada pada index paling akhir. Kita juga akan menggunakan chown agar log hanya dapat dibaca user.


hasil run:

Agar aggregate_minutes_to_hourly_log.sh dapat dijalankan tiap jam, kita dapat menambahkan line berikut pada crontab:
0 * * * * /{path aggregate_minutes_to_hourly_log.sh}


