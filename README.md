Random-Image-Script
===================

script which create JPEG thumbnails

```
#!/bin/bash

####################################################################################
#                                                                                  #
# Author https://github.com/admiralsmaster/                                        #
#                                                                                  #
# Script creates thumbnails of pictures or soft links to the original pictures.    #
# The source pictures are all found picures from an other folder (recursivly).     #
#                                                                                  #
# You can run this script via crontab.                                             #
# i.e. 5 1 * * * <path>/images/random/random.sh                                    #
#                                                                                  #
#                                                                                  #
# Script is for Joomlas random image plugin. This plugin can only use pictures     #
# from one folder without subfolders and use the original pictures without         #
# optimizing the size.                                                             #
# Disadvantage of soft lnking: new (referenced) picures have new path:             #
#                              no cache of original picture in browser.            #
# Disadvabtage of thumbnails:  more CPU usage while creation,                      #
#                              needs imagemagick                                   #
#                              sudo apt-get install imagemagick                    #
#                                                                                  #
####################################################################################


# USER	
MAIL_USER="${USER}@${HOSTNAME}"

# absolute path for target folder to create links
F=/var/www/vhosts/kuechler.info/httpdocs/6/images/random

# relative path from target folder to the folder to search pictures
T=../stories/linda

# true: create thumbnails with imagemagick
# false: create soft links 
THUMBNAILS=true

# Size of thumbnails
# Format see manual page of convert 
THUMBNAILS_SIZE=190x190

# Quality of thumbnails (0...100)
# Format see manual page of convert 
THUMBNAILS_QUALITY=60



# change seperator to newline: for pictures with spaces into name
IFS='
'

LOG_ERROR="LOG_ERROR"
LOG_WARN="LOG_WARN"
LOG_INFO="LOG_INFO"

# log to syslog
function log() {
	local now=$(date);
        local message="${now} Random Pics: $1 "
        local level=$2

        if [[ "${level}" == "${LOG_ERROR}" ]]; then
                logger -i -p cron.err $message  
		echo $message | mail -s "Error Message from ${HOSTNAME}" ${MAIL_USER}
		exit 1
        elif [[ "${level}" == "${LOG_WARN}" ]]; then
                logger -i -p cron.warning $message
        else 
                logger -i -p cron.info $message
        fi      
}



log "start creation at ${F}" ${LOG_INFO}

# find new folder a or b
if [[ -d "${F}/a" ]]; then
	NEW="${F}/b"
	OLD="${F}/a"
else 
	NEW="${F}/a"
	OLD="${F}/b"
fi

# clean new and create new
if [[ -d $NEW ]]; then
	rm -rf $NEW || log "Cannot remove folder ${NEW}" ${LOG_ERROR}
fi
mkdir $NEW || log "Cannot create new folder ${NEW}" ${LOG_ERROR}




# find files with picture ending, don't use from thumbnail and v_sign plugin
found=$(find ${F}/${T} -type f | grep -i -E '.jpg$|.jpeg$|.png$|.gif$'| grep -v thumbnail|grep -v vsig_)

# loop tru files
i=0;
for pat in ${found[*]}; do
	# calculate new path: remove prefix from absolute path, remove /, spaces, double dots, and replace double underline (is replacement) with one underline
	newPath=${NEW}/$(echo ${pat} | sed "s|${F}||" | sed 's|/|_|g' | sed 's|\s|_|g' | sed 's|\.\.|_|g' | sed 's|_\+|_|g' );
	if [[ ${THUMBNAILS} ]]; then
		/usr/bin/nice -n 19 /usr/bin/convert -thumbnail ${THUMBNAILS_SIZE} -quality ${THUMBNAILS_QUALITY} ${pat} ${newPath} || log "Cannot create thumbnail to new folder" ${LOG_ERROR}
	else
		ln -s ${pat} ${newPath} || log "cannot create softlink ${path} -> ${newPath}" ${LOG_ERROR}
	fi
	let i+=1;
done




# create "current" softlink if not exists
LINK_CUR="${F}/current"
if [[ ! -L "${LINK_CUR}" ]]; then
	ln -s ${OLD} ${LINK_CUR} || log "Cannot create softlink to current folder" ${LOG_ERROR}
	log "Create current link to ${OLD}"
fi

# create "new" softlink if not exists
LINK_NEW="${F}/new"
if [[ ! -L "${LINK_NEW}" ]]; then
	ln -s ${NEW} ${LINK_NEW} || log "Cannot create softlink to new folder" ${LOG_ERROR}
	log "Create new link to ${NEW}"
fi




# switch and remove old folder
mv -T ${LINK_NEW} ${LINK_CUR}  || log "Cannot switch to new folder" ${LOG_ERROR}
rm -rf ${OLD}


log "Finished creation with ${i} pictures at ${NEW}" ${LOG_INFO}

# EOF
```
