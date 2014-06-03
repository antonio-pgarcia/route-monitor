#!/usr/bin/bash 

#--------------------------------------------------------------------------------
# @Route Startup Script 2.0
# HolisticView - 2007
#--------------------------------------------------------------------------------

#--------------------------------------------------------------------------------
# USED PROCS
#--------------------------------------------------------------------------------
AWK=/usr/bin/awk
GREP=/usr/bin/grep
PING=/usr/sbin/ping
ROUTE=/usr/sbin/route
PKILL=/usr/bin/pkill
CKSUM=/usr/bin/cksum
LOGGER=/usr/bin/logger

#--------------------------------------------------------------------------------
# DEFAULT ROUTE CONFIG FILE
#--------------------------------------------------------------------------------
CONFIG=/etc/inet/routes.conf

#--------------------------------------------------------------------------------
# GLOBAL VARIABLES
#--------------------------------------------------------------------------------
declare -a GLOBAL_TABLE
declare -a GLOBAL_HAROUTES
declare -a GLOBAL_MONITOR
declare -i GLOBAL_TIMEOUT=10

LOGGER_PRIORITY="daemon.notice"
PROC_NAME="rt-mon"

#--------------------------------------------------------------------------------
# @SYSLOG
# 
# @param $1: MESSAGE
# @return
# @desc Logs a message to syslog 
#--------------------------------------------------------------------------------
SYSLOG()
{	
	MESG=$1
	${LOGGER} -p ${LOGGER_PRIORITY} -t ${PROC_NAME} ${MESG}
}

#--------------------------------------------------------------------------------
# @HARAKIRI
# 
# @param 
# @return
# @desc Stops rt-mon process
#--------------------------------------------------------------------------------
HARAKIRI()
{
	SYSLOG "shutting down rt-mon"
	${PKILL} -9 ${PROC_NAME}
}

#--------------------------------------------------------------------------------
# @LOADCONFIG
# 
# @param
# @return
# @desc Loads GLOBAL_TABLE with routes.conf file values
#--------------------------------------------------------------------------------
LOADCONFIG()
{
	declare -i INDEX=0
	while read LINE
	do
		if [ "${LINE}" != "" ]
		then
			GLOBAL_TABLE[${INDEX}]=${LINE}
			INDEX=${INDEX}+1
		fi
	done < ${CONFIG}
}

#--------------------------------------------------------------------------------
# @ADDROUTE
# 
# @param $1 DESTINATION
# @param $2 MASK
# @param $3 GATEWAY
# @return
# @desc 
#--------------------------------------------------------------------------------
ADDROUTE() 
{
	DEST=$1
	MASK=$2
	GATW=$3
	if [ ${MASK} != "255.255.255.255" ]
	then
		${ROUTE} add -net ${DEST} -netmask ${MASK} ${GATW} > /dev/null
	else
		${ROUTE} add -host ${DEST} ${GATW} > /dev/null
	fi
}

#--------------------------------------------------------------------------------
# @UPDROUTE
# 
# @param $1 DESTINATION
# @param $2 MASK
# @param $3 GATEWAY
# @return
# @desc 
#--------------------------------------------------------------------------------
UPDROUTE()
{
	DEST=$1
	MASK=$2
	GATW=$3
	${ROUTE} change -net ${DEST} -netmask ${MASK} ${GATW} > /dev/null
}

#--------------------------------------------------------------------------------
# @DELROUTE
# 
# @param $1 DESTINATION
# @param $2 MASK
# @param $3 GATEWAY
# @return
# @desc 
#--------------------------------------------------------------------------------
DELROUTE()
{
	DEST=$1
	MASK=$2
	GATW=$3
	${ROUTE} delete -net ${DEST} -netmask ${MASK} ${GATW} > /dev/null
}

#--------------------------------------------------------------------------------
# @CHKGATEWAY
# 
# @param $1 GATEWAY
# @return RC=0: gateway is up; RC=1: gateway is down
# @desc 
#--------------------------------------------------------------------------------
CHKGATEWAY()
{
	GATEWAY=${1}
	${PING} ${GATEWAY} > /dev/null; RC=$?
	return $RC
}

#--------------------------------------------------------------------------------
# @DROPROUTES
# 
# @param 
# @return 
# @desc Drops all routes before adds new ones
#--------------------------------------------------------------------------------
DROPROUTES()
{
	for LINE in "${GLOBAL_TABLE[@]}"
	do
		DEST=$( echo ${LINE} | ${AWK} '{print $1}' )
		MASK=$( echo ${LINE} | ${AWK} '{print $2}' )
		GATW=$( echo ${LINE} | ${AWK} '{print $3}' )
		BACK=$( echo ${LINE} | ${AWK} '{print $4}' )
		DELROUTE ${DEST} ${MASK} ${GATW}
		
		if [ "${BACK}" != "" ]
		then
			DELROUTE ${DEST} ${MASK} ${BACK}
		fi
	done
}

#--------------------------------------------------------------------------------
# @ADDROUTES
# 
# @param 
# @return 
# @desc Adds all routes from the GLOBAL_TABLE array
#--------------------------------------------------------------------------------
ADDROUTES()
{
	for LINE in "${GLOBAL_TABLE[@]}"
	do
		DEST=$( echo ${LINE} | ${AWK} '{print $1}' )
		MASK=$( echo ${LINE} | ${AWK} '{print $2}' )
		GATW=$( echo ${LINE} | ${AWK} '{print $3}' )
		BACK=$( echo ${LINE} | ${AWK} '{print $4}' )
	
		ADDROUTE ${DEST} ${MASK} ${GATW}
	
		if [ "${BACK}" != "" ]
		then
			GLOBAL_HAROUTES[ $(echo ${DEST}|${CKSUM}|${AWK} '{print $1}') ]=${LINE}
			GLOBAL_MONITOR[ $(echo ${GATW}|${CKSUM}|${AWK} '{print $1}') ]="${GATW} ${BACK}"
		fi 
	done
}

#--------------------------------------------------------------------------------
# @UPDROUTES
# 
# @param $1: GATEWAY
# @return 
# @desc modify using the GLOBAL_HAROUTES array 
#--------------------------------------------------------------------------------
UPDROUTES()
{
	NEWGATEWAY=${1}
	for LINE in "${GLOBAL_HAROUTES[@]}"
	do
		DEST=$( echo $LINE | ${AWK} '{print $1}' )
		MASK=$( echo $LINE | ${AWK} '{print $2}' )
		UPDROUTE ${DEST} ${MASK} ${NEWGATEWAY}
	done
}

#--------------------------------------------------------------------------------
# @UPDROUTES
# 
# @param $1: PRIMARY GATEWAY
# @param $2: SECONDARY GATEWAY
# @return STATE: 1: primary up; 2: secondary up
# @desc 
#--------------------------------------------------------------------------------
STATE()
{
	PRI=${1}
	SEC=${2}

	STATE=1
	
	CHKGATEWAY ${GATW}; STATUS=$?
	if [ ${STATUS} -ne 0 ]
	then
				
		CHKGATEWAY ${BACK}; STATUS=$?
		if [ ${STATUS} -eq 0 ]
		then
			STATE=2
		fi
	fi
	return ${STATE}
}

#--------------------------------------------------------------------------------
# @MONITOR
# 
# @param 
# @return 
# @desc Dead gateway monitoring 
#--------------------------------------------------------------------------------
MONITOR()
{
	STATE_SAVE=1
	while true
	do
		for LINE in "${GLOBAL_MONITOR[@]}"
		do
			GATW=$( echo $LINE | ${AWK} '{print $1}' )
			BACK=$( echo $LINE | ${AWK} '{print $2}' )
		
			STATE ${GATW} ${BACK}; STATE=$?
			if [ ${STATE} -ne ${STATE_SAVE} ]
			then
				STATE_SAVE=${STATE}
				if [ ${STATE} -eq 1 ]
				then
					SYSLOG "gw ${GATW} up"  
					UPDROUTES ${GATW}
				fi
				if [ ${STATE} -eq 2 ]
				then
					SYSLOG "gw ${GATW} down"  
					UPDROUTES ${BACK}
				fi
			fi
		done
		sleep ${GLOBAL_TIMEOUT}
	done
}

#--------------------------------------------------------------------------------
# MAIN PROC
#--------------------------------------------------------------------------------

case "$1" in
'start')
	SYSLOG "starting rt-mon"
	LOADCONFIG 
	DROPROUTES
	ADDROUTES
	MONITOR &
;;

'stop')	
	HARAKIRI
;;

'remove')	
	SYSLOG "flushing routes"
	LOADCONFIG 
	DROPROUTES	
;;
esac

