```sh
#!/bin/bash
[[ $(id -u) -eq 0 ]] || exec sudo /bin/bash -c "$(printf '%q ' "$BASH_SOURCE" "$@")"
progname=$(basename $0)
quiet=false
no_dry_run=false
while getopts ":qn" opt
do
    case "$opt" in
      q)
          quiet=true
          ;;
      n)
          no_dry_run=true
          ;;
      ?)
          echo "unexpected option ${opt}"
          echo "usage: ${progname} [-q|--quiet]"
          echo "    -q: no output"
          echo "    -n: no dry run (will remove unused directories)"
          exit 1
          ;;
    esac
done
shift "$(($OPTIND -1))"

[[ ${quiet} = false ]] || exec /bin/bash -c "$(printf '%q ' "$BASH_SOURCE" "$@")" > /dev/null

echo "Running as: $(id -un)"

progress_bar() {
    local w=80 p=$1;  shift
    # create a string of spaces, then change them to dots
    printf -v dots "%*s" "$(( $p*$w/100 ))" ""; dots=${dots// /.};
    # print those dots on a fixed-width space plus the percentage etc.
    printf "\r\e[K|%-*s| %3d %% %s" "$w" "$dots" "$p" "$*";
}

cd /var/lib/docker/overlay2
echo cleaning in ${PWD}
i=1
spi=1
sp="/-\|"
directories=( $(find . -mindepth 1 -maxdepth 1 -type d | cut -d/ -f2) )
images=( $(docker image ls --all --format "{{.ID}}") )
total=$((${#directories[@]} * ${#images[@]}))
used=()
for d in "${directories[@]}"
do
    for id in ${images[@]}
    do
        ((++i))
        progress_bar "$(( ${i} * 100 / ${total}))" "scanning for used directories ${sp:spi++%${#sp}:1} "
        docker inspect $id | grep -q $d
        if [ $? ]
        then
            used+=("$d")
            i=$(( $i + $(( ${#images[@]} - $(( $i % ${#images[@]} )) )) ))
            break
        fi
    done
done
echo -e "\b\b " # get rid of spinner
i=1
used=($(printf '%s\n' "${used[@]}" | sort -u))
unused=( $(find . -mindepth 1 -maxdepth 1 -type d | cut -d/ -f2) )
for d in "${used[@]}"
do
    ((++i))
    progress_bar "$(( ${i} * 100 / ${#used[@]}))" "scanning for unused directories ${sp:spi++%${#sp}:1} "
    for uni in "${!unused[@]}"
    do
        if [[ ${unused[uni]} = $d ]]
        then
            unset 'unused[uni]'
            break;
        fi
    done
done
echo -e "\b\b " # get rid of spinner
if [ ${#unused[@]} -gt 0 ]
then
    [[ ${no_dry_run} = true ]] || echo "Could remove:  (to automatically remove, use the -n, "'"'"no-dry-run"'"'" flag)"
    for d in "${unused[@]}"
    do
        if [[ ${no_dry_run} = true ]]
        then
            echo "Removing $(realpath ${d})"
            rm -rf ${d}
        else
            echo " $(realpath ${d})"
        fi
    done
    echo Done
else
    echo "All directories are used, nothing to clean up."
fi
```