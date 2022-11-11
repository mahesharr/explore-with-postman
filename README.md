# explore-with-postman
A forked copy of Amber Race's Test Automation University course code, to accompany Beth Marshall's Test Automation University Course, "API Test Automation With Postman".

You can discover more of Amber's amazing code for API exploratory testing on her github page here: https://github.com/ambertests

import sys
import paramiko
from paramiko.ssh_exception import SSHException, AuthenticationException
import os
import json
import pymysql
import argparse
import re
import subprocess


def init_parser():
    """Parsing input parameters to database."""
    parser = argparse.ArgumentParser()
    parser.add_argument('--ip_address', help='Robot ip', required=False)
    parser.add_argument('--user', help='user', required=False)
    args = parser.parse_args()
    return args


def open_connection(args):
    """open connection to robot"""
    try:
        print("opening a connection")
        client = paramiko.client.SSHClient()
        client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        client.connect(args.ip_address, username=args.user, password='')
        return client
    except(AuthenticationException, SSHException):
        sys.exit("connection timed out")


def get_robot_serial(client):
    cmd = " /usr/local/bin/roboctrl -R m  | sed '2!d' "
    _stdin, _stdout, _stderr = client.exec_command(str(cmd))
    output = _stdout.read()
    s = re.findall(r"(?<=').*?(?=\\)", str(output))
    serial_number = s[0]
    # for getting date and time on host machine
    command = 'date -u --iso-8601=sec'
    _stdin, _stdout, _stderr = client.exec_command(command)
    date = _stdout.read()
    s = re.findall(r"(?<=').*?(?=\\)", str(date))
    date = s[0]
    robot_serial_date = {"serial_number": serial_number, "date": date}
    return robot_serial_date


def create_log_path(robot_serial, config_file):
    parent_cwd= os.getcwd()
    print("i want  ...",parent_cwd)
    config_read = open(config_file)
    data = json.load(config_read)
    log_path = data['log_path']
    config_read.close()
    if not os.path.exists(log_path):
        os.makedirs(log_path)
    if not os.path.exists(robot_serial["serial_number"]):
        os.makedirs(parent_cwd + "//" + log_path + "//" + robot_serial["serial_number"])
        os.chdir(parent_cwd + "//" + log_path + "//" + robot_serial["serial_number"])

    if not os.path.exists(parent_cwd + log_path + "//" + robot_serial["serial_number"] + "//" + robot_serial["date"]):
        os.makedirs(robot_serial["date"])
        os.chdir(robot_serial["date"])
        cwd = os.getcwd()
        return cwd, parent_cwd


def get_robot_logs(args):
    # /user/black_box
    if not os.path.exists("black_box"):
        print("black box")
        os.makedirs("black_box")
        source_path = ":/user/black_box/*"
        save_path = " black_box"
        # os.system("scp -r root@" + ip_address + ":/user/black_box/* black_box")
        os.system("scp -r root@" + args.ip_address + source_path + save_path)
    # /var/log/journal/
    if not os.path.exists("journal"):
        print("journal")
        os.makedirs("journal")
        source_path1 = ":/var/log/journal/*"
        save_path1 = " journal"
        os.system("scp -r root@" + args.ip_address + source_path1 + save_path1)


def get_data_db(args, config_file, parent_cwd):
    print("get_data... ", os.getcwd())
    config_read = open(parent_cwd + "//" + config_file)
    data = json.load(config_read)
    hostname = data["database_server"]["hostname"]
    user = data["database_server"]["user"]
    password = data["database_server"]["password"]
    dbname = data["database_server"]["dbname"]

    dbconnection = pymysql.connect(host=hostname, user=user, password=password, db=dbname,
                                   charset="utf8mb4", cursorclass=pymysql.cursors.DictCursor)
    cursor = dbconnection.cursor()
    _SQL = ("select * from robot_details where ip_address =\'" + args.ip_address + "\'")
    cursor.execute(_SQL)
    result = cursor.fetchall()
    return result


def create_metafile(args, config_file, parent_cwd):
    result = get_data_db(args, config_file, parent_cwd)
    cwd = os.getcwd()
    for row in result:
        jsonobj = json.dumps(row, indent=4)
    y = json.loads(jsonobj)
    with open(cwd + "//robotmeta.data", "w") as f:
        f.write(str(jsonobj))
    f.close()


if __name__ == '__main__':
    args = init_parser()
    client = open_connection(args)
    robot_serial_date = get_robot_serial(client)
    parent_cwd, cwd = create_log_path(robot_serial_date, 'config.json')
    get_robot_logs(args)
    create_metafile(args, 'config.json', parent_cwd)
