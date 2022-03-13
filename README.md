<h1># Docker-Dasar</h1>


#Docker Image
-> docker image ls => melihat image yang ada di machine
-> docker image pull namaimage:tag => mendownload image dari docker registry beserta versi dari image nya
-> docker image rm namaimage:tag => menghapus docker image

#Docker Container (Manajemen Container)
-> docker container ls -a => melihat container yang ada di docker kita
-> docker container ls => melihat container yang sedang berjalan
-> docker container create --name namacontainer nameimage:tag => membuat container baru
-> docker container start containerid/namacontainer => menjalankan container yang sudah dibuat
-> docker container stop containerId/namacontainer => menghentikan container yang jalan
-> docker container rm containerId/namacontainer => menghapus sebuah container

#Container Log
-> docker container logs containerId/containername => melihat log aplikasi di container kita
-> docker container logs -f containerId/containerName => melihat log secara real time

#Container Exec
untuk masuk kedalam container, kita bisa mencoba mengeksekusi bash script yang 
terdapat didalam container dengan bantuan container exec
-> docker container exec -i -t containerId/containername /bin/bash
	-i adalah argumen interaktif, menjaga input tetap aktif
	-t adalah argument untuk alikasi pseudo-TTY (terminal akses)
	dan /bin/bash contoh kode program yang terdapat didalam container

#Container Port
Berguna untuk membuat container kita bisa diakses dari luar container
untuk melakukannya kita harus membuat port nya dari awal saat container dibuat
-> docker container create --name namacontainer --publish porthost:portcontainer image:tag
	--publish juga bisa disingkat -p
	bisa juta port forwarding lebih dari satu dengan menambahkan dua kali --publish

#Container Environment Variable (env)
biasanya untuk username dan password 
-> docker container create --name namacontainer --env KEY="value" --env KEY2="value" image:tag

#Container Stats
melihat resource yang digunakan oleh docker
-> docker container stats

#Docker Container Resource Limit
Memory
- kodenya adalah --memory diikuti dengan angka memorynya
	b(bytes), k(kilobytes), m(megabytes), g(gigabytes) : misal 100m artinya 100megabytes
CPU
- kodenya adalah --cpus, jika kita sets dengan nilai 1.5 artinya 1 dan setengah cpu core
-> docker container create --name smallnginx --publish 8081:80 --memory 100m --cpus 0.5 nginx:latest

#Bind Mounts
merupakan kemampuan melakukan mounting(sharing) file atau folder yang terdapat di sistem host ke container yang terdapat di docker
untuk melakukan mounting kita bisa menggunakan parameter --mount ketika kita membuat containernya
parameter mount:
- type = tipe mount, mount, atau volume
- source = lokasi file atau folder di sistem host
- destination = lokasi file atau folder di container
- readonly = jika ada, maka file atau folder hanya bisa dibaca di container, tidak bisa ditulis
-> docker container create --name namacontainer --mount "type=bind, source=folder, destination=folder, readonly" image:tag
-> docker container create --name mongodata --publish 27018:27017 --mount "type=bind,source=/home/fedrian/Documents/Belajar/Pemrograman/belajar-docker-dasar/mongo-data,destination=/data/db" --env MONGO_INITDB_ROOT_USERNAME=fedrian --env MONGO_INITDB_ROOT_PASSWORD=fedrian mongo:latest

#Docker Volume
Fitur bind mounts sudah ada sejak docker versi awal, di versi terbaru disarankan untuk menggunakan docker volume
docker volume mirip dengan bind mount, bedanya adalah terdapat management volume, dimana kita bisa membuat volume, melihat daftar volume, dan menghapus volume
volume sendiri bisa dianggap storage yang digunakn untuk menyimpan data , bedanya dengan bind mount data disimpan pada sistem host, sementara pada volume data di manage oleh docker
saat kita membuat container secara default semua data container akan disimpan di dalam volume
jika kita melihat docker bvolume kita akan lihat bahaw banyak volume yang terbuat, walaupun kita belum pernah membuatnya sama sekali
perintah untuk melihat daftar volume
-> docker volume ls
untuk membuat volume, printahnya:
-> docker volume create namavolume
untuk menghapus volume,pastikan containernya tidak digunakan printahnya:
-> docker volume rm namavolume


#Container Volume
sama dengan bind mount cara menggunakannya, namun dengan menggunakan type volume dan source nama volume
-> docker volume create mongodata
-> docker container create --name mongovolume --publish 27019:27017 --mount "type=volume,source=mongodata,destination=/data/db" --env MONGO_INITDB_ROOT_USERNAME=fedrian --env MONGO_INITDB_ROOT_PASSWORD=fedrian mongo:latest
-> docker container rm namacontainer

#Backup Volume
sampai saat ini, tidak ada cara otomatis untuk melakukan backup volume yang sudah kita buat
namun kita bisa memanfaatkan container untuk melakukan backup data yang ada di volume kedalam archive seperti zip atau tar.gz
tahapan=
	matikan container yang menggunakan volume yang ingin kita buat
	buat container baru dengan dua mount, volume yang ingin kita backup, dan bind mount folder dari sistem host
	lakukan backup menggunakan container dengan cara meng-archive isi volume, dan simpan di bind mount folder
	isi file backup sekarang ada di folder sistem host
	delete container yang kita gunakan untuk melakukan backup

-> docker container stop mongovolume
-> mkdir backup
	 /home/fedrian/Documents/Belajar/Pemrograman/belajar-docker-dasar/backup
-> docker container create --name nginxbackup --mount "type=bind,source=/home/fedrian/Documents/Belajar/Pemrograman/belajar-docker-dasar/backup,destination=/backup" --mount "type=volume,source=mongodata,destination=/data" nginx:latest
-> docker container start nginxbackup
-> docker container exec -i -t nginxbackup /bin/bash
cek folder data dan backup nya
->tar cvf /backup/backup.tar.gz /data
-> docker container stop nginxbackup
-> docker container rm nginxbackup
-> docker container start mongovolume

#Menjalankan container backup secara langsung 
- kita bisa menggunakan perintah run untuk menjalankan perintah di container dan gunakan parameter --rm untuk melakukan otomatis remove container setelah perintahnya selesai berjalan
-> docker image pull ubuntu:latest  
-> docker container stop containeryangingindibackup
-> docker container run --rm --name ubuntu --mount "type=bind,source=/home/fedrian/Documents/Belajar/Pemrograman/belajar-docker-dasar/backup,destination=/backup" --mount "type=volume,source=mongodata,destination=/data" ubuntu:latest tar cvf /backup/backup-lagi.tar.gz /data
-> docker container start mongovolume

#Restore Volume
- setelah melakukan backup volume kedalam file archive, kita bisa menyimpan file archive backup tersebut ke tempat yang lebih aman, misal ke cloud storage
- sekarang kita akan coba melakukan restore data backup ke volume baru, untk memastikan backup yang kita lakukan tidak corrupt
=> Tahapan
	- buat volume baru untuk lokasi restore data backup
	- buat container baru dengan dua mount, volume baru untuk restore backup, dan bind mount folder dari sistem host yang berisi file backup
	- lakukan restore menggunakan container dengan cara mengextract isi backup file kedalam volume
	- isi file backup sekarang sudah di restore ke volume
	- delete container yang kita gunakan untuk melakukan restore
	- volume baru yang berisi data backup siap diguanakan oleh container baru
-> docker volume create mongorestore
-> docker container run --rm --name ubunturestore --mount "type=bind,source=/home/fedrian/Documents/Belajar/Pemrograman/belajar-docker-dasar/backup,destination=/backup" --mount "type=volume,source=mongorestore,destination=/data" ubuntu:latest bash -c "cd /data && tar xvf /backup/backup-lagi.tar.gz --strip 1"
-> docker container create --name mongorestore --publish 27020:27017 --mount "type=volume,source=mongorestore,destination=/data/db" --env MONGO_INITDB_ROOT_USERNAME=fedrian --env MONGO_INITDB_ROOT_PASSWORD=fedrian mongo:latest

#Docker Network
- saat kita membuat container di docker, secara default container akan saling terisolasi satu sama lain, jadi jika kita mencoba memanggil antara container, bisa dipastikan bahwa kita tidak akan bisa melakukannya
- Docker memiliki fitur network yang bisa digunakan untuk membuat jaringan didalam docker
- dengan menggunakan network, kita bisa mengkoneksikan container dengan container lain dalam satu network yang sama
- jika beberapa conatiner terdapat pada satu network yang sama, maka secara otomatis container tersebut bisa saling berkomunikasi

=>Network Driver
- saat kita membuat network di docker, kita perlu menentukan driver yang ingin kita gunakan, ada banyak driver yang bisa kita gunakan, tapi kadang ada syarat sebuah driver network bisa kita gunakan
- bridge, yaitu driver yang digunakan untuk membuat network secara virtual yang memungkinkan container yang terkoneksi di bridge network yang sama saling berkomunikasi
- host, yaitu driver yang digunakan untuk membuat network yang sama dengan sistem host. host hanya jalan di docker linux, tidak bisa digunakan di mac atau windows
- none, yaitu driver yang membuat network yang tidak bisa berkomunikasi

-> docker network ls => melihat network
-> docker network create --driver namadriver namanetwork => membuat network baru
-> docker network rm namanetwork => menghapus container, tidak bisa dihapus jika sudah digunakan container, kita harus menghapus container nya terlebih dahulu

#Container Network
- setelah kita membuat network, kita bisa menambahkan container ke network
- container yang terdapat didalam network yang sama bisa saling berkomunikasi (tergantung jenis driver networknya)
- container bisa mengakses container lain dengan menyebutkan hostname dari container nya, yaitu nama container nya

=>Membuat container dengan network
- untuk menambahkan container ke network, kita bisa menambahkan perintah --network ketika membuat container
-> docker container crate --name namacontainer --network namanetwork image:tag
Contoh:
-> docker network create --driver bridge mongonetwork
-> docker container create --name mongodb --network mongonetwork --env MONGO_INITDB_ROOT_USERNAME=fedrian --env MONGO_INITDB_ROOT_PASSWORD=fedrian mongo:latest
-> docker image pull mongo-express:latest
-> docker container create --name mongodbexpress --network mongonetwork --publish 8081:8081 --env ME_CONFIG_MONGDB_URL="mongodb://fedrian:fedrian@mongodb:27017/" mongo-express:latest
-> docker container start mongodb
-> docker container start mongoexpress

=>Menghapus Container dari Network
- jika diperlukan, kita juga bisa menghapus container dari network dengan perintah:
-> docker network disconnect nananetwork namacontainer

=>Menambahkan Container ke Network
- jika containernya sudah terlanjur dibuat, kita juga bisa menambahkan container yang sudah dibuat ke network dengan perintah:
-> docker network connect namanetwork namacontainer

#Inspect
- setelah kita mendownload image, atau membuat network, volume dan container, kadang kita ingin melihat detail dari tiap hal tersebut
- docker memiliki fitur bernama inspect, yang bisa kita gunakan di image, container, volume, dan network
- dengan fitur ini, kita bisa melihat detail dari tiap hal yang ada di docker

=> Menggunakan Inspect
- melihat detail dari image, gunakan		=> docker image inspect namaimage
- melihat detail dari container, gunakan	=> docker image inspect namacontainer
- melihat detail dari volume, gunakan		=> docker image inspect namavolume
- melihat detail dari network, gunakan		=> docker image inspect namanetwork


#Prune
- saat kita menggunakan docker, kadang ada kalanya kita ingin membersihkan hal-hal yang sudah tidak digunakan lagi di docker, misal container yang sudah di stop, image yang tidak digunakan oleh container, atau volume yang tidak digunakan oleh container
- fitur untuk membersihkan secara otomatis di docker bernama prune
- hampir di semua perintah docker mendukung prune

=>Perintah Prune
- untuk menghapus semua container yang suda stop, gunakan				=> docker container prune
- untuk menghapus semua image yang tidak digunakan container, gunakan			=> docker image prune
- untuk menghapus semua network yang tidak digunakan container container, gunakan 	=> docker network prune
- untuk menghapus semua volume yang tidak digunakan container, gunakan			=> docker volume prune
- atau kita bisa menggunakan satu perintah untuk menghapus container, network dan image yang sudah tidak digunakan menggunakan perintah	=> docker system prune











