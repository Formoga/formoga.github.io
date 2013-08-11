---
layout: default
title: Setup TeamCity CI
permalink: SetupTeamCityCI
---

# Overview #

團隊中的開發除了Unity的專案外，會開始陸續展開其它的專案，而這些軟體專案，若是能夠用自動化的方式去做建置，則可大幅度的提高建置過程中的品質。因此選用Continuous Integration Sever(CIS)勢必是不可缺少的。考量到團隊內的資源運用，會優先選擇Open Source或是在一定量下的使用是免費或是小額的方案。而符合這樣考量的，則有[Jenkins][100010](Open Source)、[TeamCity][100020](一定使用量下免費方案)或是[Bamboo][100030](小額付費方案)。在Jenkins的主要撰寫人加入[CloudBees][100040]後，總覺得Jenkins的Open Source版已沒有太大的改進，而Bamboo的小額使用下，限制異常的多。所以最後決定採用TeamCity，不但版本的更新速度快、介面乾淨，最主要的是目前的License，非常替小型團隊著想。這也是商用版中，最符合目前團隊開發需求的CIS。

一但選擇了是用TeamCity當做CI，依據以往使用Jenkins的經驗，展開了架設的動作。團隊中目前有一專門拿來放置檔案的PC，雖然選擇了還在開發中的Windows 8.1 Preview當做OS，但其穏定的核心是令人放心的，且就以現階段的資源配置，若是灌Linux(Ubuntu)反而不適當，有些專案一定要在.Net上開發和建置，就連Mono都沒有辦法應付這些專門以.Net為主寫出來的專案。若是真的有Linux的需求，團隊內現在的Mac是可以應付的，若是一定要用到Linux的環境，Virtual Machine非常普及的當下，在VM中灌個Linux不是件困難的事。實際上，TeamCity的Server就是跑在Windows這台機子裡的VM中。

不論是TeamCity的官方建議，或是網路上有著架設CI Server經驗的開發者，都一致認為CI的Server和Build Agent是要分開的。大概說明一下，於多數CI服務或是軟體概念中，會分成資料滙集的部份和實際展開建罝工作二個部份。在資料滙集中，包含了誰可以用，怎麼用等資料。但正式要開始建置工作時，往往會需要將建置的任務配置到某個電腦(OS)，也就是一至多個跑在特定OS下的Agent。為什麼會有這樣的區別？就拿建置iOS App(或是Mac OS App)的專案來說，一定要從XCode裡建置，而XCode只有在Mac OS裡才能合法的使用。所以，若是今天的專案是iOS App，則一定要將這項建置的任務導到Mac上的Agent做執行，但資料的滙集(專案相關的程式碼等)，一開始是不用一定要在Mac上滙集的。

因此在這樣的規劃下，Windows本身會成為Build Agent，但在運行於VM裡的Linux反而是主要的Server。實際上，TeamCity是提供多種版本的，不論是Linux, Windows, Mac或是放在Java EE Web Server的版本都可取得，但配合現有的資源和考量，在Windows裡運行另一個跑著VM的Windows是沒有必要的，甚至是跑有GUI介面的Linux都有些多餘。能夠省下不必要的記憶體配置留給後續建置工作，也是規劃上考量的重點之一。

VM的選擇，在Windows的環境下，主要分為二套，Open Source陣營的[Virtual Box][100050]和商用但有免費使用方案的[VMWare][100060]。就這二個不同的VM來說，哪一個都可行，在官方的功能上，也都沒有特別提到支援Windows 8.1 Preview，所以隨便選一個可以運作的情況下，選到了Virtual Box，而運行上沒有問題，就最後選擇了用Virtual Box當在VM軟體，而Linux的Distro就選擇了目前商業支持度最高，每年也有二次重大改版的Ubuntu，不需要多餘的UI干擾，選擇了[Server 64 bit版本][100070]。

一如往常的在Virtual Box裡照著安裝精靈一步一步的往下走，記得將Ubuntu的ISO檔放到Virtual CD/DVD裝置上，關掉Audio，而網路的設定先用預設的NAT。放個2G到4G的記憶體，而Hard Disk的容量，在本機隨便都是1TB起跳的情況下，直接配置100GB都可。而進入到Ubuntu的安裝時，全部都走預設的即可。完成Ubuntu的安裝後，接下來就正式的進入了設定TeamCity Server的階段。

基於安全考量(雖然不一定有這樣的必要)，會額外創建TeamCity專用的使用者帳號，參考[這篇文章][100100]後可以輕易的完成，是不是系統帳號倒無所謂，以目前團隊的配置規模，考量到太多的安全性細節並不一定是最好的選擇。且TeamCity的官方文件裡，沒有強烈要求TeamCity是否要以管理者權限的使用者在Linux中啓用，所以額外創建的使用者是只有一般的權限，在安全性的考量下，還算是安全的。但很多開發相關的軟體設定、拿取，則需要透過原先的管理者才能進行。

按照TeamCity官方的安裝文件，Java Run Time(JRE)是一定要裝的，且推薦的版本還是1.6版。如果該Server會在執行時直接開啓一個Build Agent，則需要使用Java Development Kit(JDK)，而不是JRE就可運作的。而預設的Database也僅供測試使用，建議是換成MySql，用起來效率較高。而接下來會進入到實際指令的操作、設定。

在Ubuntu安裝Java，用PPA的機制後，變得是很簡單的，在具有管理權限的使用者帳號裡，先輸入移除內建OpenJDK package的指令

    sudo apt-get purge openjdk*

而後再將PPA加入

    sudo add-apt-repository ppa:webupd8team/java

待Update後

    sudo apt-get update

即可依照所需的Java版本輸入安裝指令

    # Java 6
    sudo apt-get install oracle-java6-installer
    # Java 7
    sudo apt-get install oracle-java7-installer
    # Java 8
    sudo apt-get install oracle-java8-installer

考量到日後版本控管有需要用到Git的服務，也可以拿取[Peter van der Does][100200]所包裝好的git

    sudo add-apt-repository ppa:pdoes/ppa
    sudo apt-get update
    sudo apt-get install git

到此，如果不考量MySql的安裝，是已經可以展開TeamCity的佈署。

轉至專門替TeamCity Server所建立的使用者帳號，在此帳號中下載TeamCity的壓縮檔，目前最新的版號為8.0.2版，故打入`wget`拿取後再用`tar`解押

    wget http://download.jetbrains.com/teamcity/TeamCity-8.0.2.tar.gz
    tar -xzvf TeamCity-8.0.2.tar.gz

接著可於解壓縮後，直接切換目錄到`TeamCity/bin`，從[官方的文件][100300]中可了解到，若是要直接啓動Server，則可輸入

    ./teamcity-server.sh start

如果要啓動Server和一個Build Agent，則輸入

    ./runAll.sh

然而，直接打入`./runAll.sh`會產生錯誤，會需要額外在`TeamCity`和`bin`同層的目錄裡另外產生名為`logs`的目錄，但以`./teamcity-server.sh`方式執行時，會自動產生logs目錄。如同前面提及的，不會讓Server和Build Agent放在同一台(概念上)電腦裡。故只會獨自啓動Server，而另行將Build Agent放到其它的電腦上。

由其它電腦網頁連線進此Server，以Web UI行式進行操控，因為是Linux Server，所以會開在8111預設的Port上，因此連線時要額外放入Port的資訊(192.168.x.x:8111)。連線展開時，會因為沒有TeamCity Data Directory的情況下，額外產生`.BuildServer`這個目錄資料。而一但此目錄產生後，才能夠進行更換External Database(像是MySql、Postgresql等)的動作。

如果先前在裝Ubuntu時，沒有選擇額外安裝LAMP的服務，則必需切回到有管理權限的帳號進行安裝MySql的動作

    sudo apt-get install mysql-server

用此方法安裝mysql時，會先要求使用者對mysql的root進行密碼的設定。待安裝完成後，按照TeamCity的官方設定，先進入到MySql的console後，以令產生所需要用的表格，而此步驟沒有Ubuntu相關的權限要求，所以可用管理權限帳號或是切回teamcity帳號執行

    mysql -u root -p

待輸入密碼後，即可在mysql console裡以root的身份輸入下列指令

    create database <database name> default charset utf8;
    grant all privileges on <database name>.* to <user>@localhost identified by '<password>';

完成此步驟後，MySql部份就設定完成，接下來，又回到TeamCity裡的設定。

[官方文件][100400]裡有提到礙於License的協定，不能將各Database的JDBC放入，要請開發者自行下載取得。而MySql的Connector放在Oracle的網站上，若用Browser拿取到還不複雜，可是由純文字的模下載時，就有些額外的步驟要處理。一樣用`wget`下載

    wget http://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.26.tar.gz/from/http://cdn.mysql.com/

但下載回來的檔案是`index.html`要自行更名成`mysql-connector-java-5.1.26.tar.gz`後，再解壓縮。解壓後，要進入目錄裡將`mysql-connector-java-5.1.26.jar`取出，放到指定的TeamCity Data Directory(預設為`.BuildServer`目錄)下的`lib/jdbc`目錄中。之後，再進入TeamCity Data Directory下的config目錄，將`database.mysql.properties.dist`複製後更名成database.properties並於此檔中進行修改

    connectionUrl=jdbc:mysql://<host>/<database name>
    connectionProperties.user=<user>
    connectionProperties.password=<password>

此處的host會放入localhost，而database name、user以及password，都按照之前在mysql console裡打進去的的放入。完成後，換到`TeamCity/bin`目錄下開始server。但在Web UI操作下，會看到沒有Database的錯誤顯示，並要求要輸入TeamServer管理者才看得到的資訊，才能繼續往下動作。而此資訊，則是放在`TeamCity/logs`目錄下的`teamcity-server.log`裡，再當下最後面那一行中會出現一串超長的數字要輸入，而輸入後才能正確進行下去。一切正常後，會被要求產生一TeamCity管理權限的帳號，待產生後，則會在Browser看到一個沒有Build Agent，沒有專案的初始化TeamCity內容頁面。

[100010]: http://jenkins-ci.org "Jenkins"
[100020]: http://www.jetbrains.com/teamcity/ "TeamCity"
[100030]: https://www.atlassian.com/software/bamboo "Bamboo"
[100040]: http://www.cloudbees.com/#slide-1 "CloudBees"
[100050]: https://www.virtualbox.org "Jenkins"
[100060]: http://www.vmware.com/ap/ "VMWare"
[100070]: http://www.ubuntu.com/download/server "Ubuntu Server"
[100100]: http://www.howtogeek.com/howto/ubuntu/add-a-user-on-ubuntu-server/ "Add a user on ubuntu server"
[100200]: http://blog.avirtualhome.com/git-ppa-for-ubuntu/ "Git PPA for Ubuntu"
[100300]: http://confluence.jetbrains.com/display/TCD8/Installing+and+Configuring+the+TeamCity+Server "Installing and Configuring the TeamCity Server"
[100400]: http://confluence.jetbrains.com/display/TCD8/Setting+up+an+External+Database "Setting up an External Database"
