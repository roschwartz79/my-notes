# Bash notes and review

## Some basics to review

### Wildcards
* A * matches any characters
* A ? matches a single character
* g* matches any file beginning with g
* Data??? matches any file beginning with Data followed by exactly 3 characters
* [:alpha:] matchse any alphabetic letter

### Basic commands
* MKDIR
    * mkdir dir1 dir2 dir3 will create 3 different directories
* cp item/dir item/dir
    * Flags:
    * -a means copy the files/dir and ALL the attributes, ownerships and permissions
    * -i means before it overwrites, it will prompt the user for confirmation to overwrite
    * -r means recursively copy dirs and their contents
    * -umeans when it copies files, only copy files that either dont exist or are newer than corresponding files
    * -v means display verbose logging when copying
