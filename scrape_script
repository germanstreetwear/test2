import logging
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from bs4 import BeautifulSoup
import requests
import firebase_admin
from firebase_admin import credentials, firestore
import time
import os
from concurrent.futures import ThreadPoolExecutor

# Logging-Konfiguration
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")

# Firebase initialisieren
try:
    # Lade die Firebase-API-Schlüssel aus einer Umgebungsvariable
    cred = credentials.Certificate(os.getenv("FIREBASE_CREDENTIALS_JSON"))
    firebase_admin.initialize_app(cred)
    db = firestore.client()
    logging.info("Erfolgreich mit Firestore verbunden.")
except Exception as e:
    logging.error(f"Fehler bei der Firebase-Initialisierung: {e}")
    db = None

# Selenium WebDriver mit Headless-Option konfigurieren
options = Options()
options.add_argument('--headless')
options.add_argument('--disable-gpu')
options.add_argument('window-size=1920x1080')
options.add_argument('--no-sandbox')

# Funktion, um den Unternehmensnamen zu extrahieren
def get_company_name(base_url, selectors):
    try:
        response = requests.get(base_url)
        soup = BeautifulSoup(response.text, 'html.parser')
        company_name_tag = soup.select_one(selectors['company_name'])
        company_name = company_name_tag.text.strip() if company_name_tag else "Unknown Company"
        logging.info(f"Unternehmensname extrahiert: {company_name}")
        return company_name
    except Exception as e:
        logging.error(f"Fehler beim Extrahieren des Unternehmensnamens: {e}")
        return "Unknown Company"

# Funktion, um alle Produkt-URLs von der Kategorieseite zu sammeln
def get_product_urls(base_url, category_url, selectors):
    try:
        response = requests.get(category_url)
        soup = BeautifulSoup(response.text, 'html.parser')
        product_urls = []
        product_blocks = soup.select(selectors['product_block'])
        
        for block in product_blocks:
            link_tag = block.select_one(selectors['product_link'])
            if link_tag and 'href' in link_tag.attrs:
                product_urls.append(base_url + link_tag['href'])
        
        product_urls = list(set(product_urls))
        logging.info(f"Gefundene Produkt-URLs: {len(product_urls)}")
        return product_urls
    except Exception as e:
        logging.error(f"Fehler beim Abrufen der Produkt-URLs: {e}")
        return []




def scrape_product_details(driver, product_url, selectors):
    try:
        driver.get(product_url)
        time.sleep(5)
        html = driver.page_source
        soup = BeautifulSoup(html, 'html.parser')
        
        product_name = soup.select_one(selectors['product_name']).text.strip() if soup.select_one(selectors['product_name']) else None
        description_elements = soup.select(selectors['product_description']) if selectors.get('product_description') else []
        stripe_description = " ".join([desc.text.strip() for desc in description_elements])
        image_div = soup.select_one(selectors['image_gallery'])
        image_urls = ["https:" + img['src'] for img in image_div.find_all('img')] if image_div else []
        
        size_elements = soup.select(selectors['size_options']) if selectors.get('size_options') else []
        sizes = {}
        for element in size_elements:
            size_name = element.get(selectors['size_value_attr'], '').strip()
            is_disabled = selectors['size_disabled_class'] in element.get('class', [])
            sizes[size_name] = not is_disabled
        
        # Preis extrahieren
        price_button = soup.select_one(selectors['price'])
        price_text = price_button.text.split('—')[-1].strip() if price_button else None
        
        if price_text is None:
            price = "sold_out"
        else:
            try:
                # Bereinige den Preistext
                price_cleaned = price_text.replace("€", "").replace(",", ".").strip()  # Komma zu Punkt
                price_cleaned = price_cleaned.replace(",-", "").replace(".", "")  # Entferne ",-" und "." am Ende
                price_cleaned = price_cleaned.replace("-", "").replace(".", "")  # Entferne ",-" und "." am Ende
                
                # Versuche die bereinigte Zahl zu konvertieren
                price = int(float(price_cleaned) * 100)  # Speichere den Preis immer in Cent
            except (ValueError, TypeError) as e:
                logging.error(f"Fehler beim Konvertieren des Preises: {price_text} - {e}")
                price = "sold_out"

        logging.info(f"Produktdetails gescraped: {product_name}")
        return {
            "name": product_name,
            "description": stripe_description,
            "images": image_urls,
            "sizes": sizes,
            "price": price,
            "url": product_url,
        }
    except Exception as e:
        logging.error(f"Fehler beim Scrapen der Produktdetails ({product_url}): {e}")
        return {}



# Funktion zum Speichern der Produktinformationen in Firestore
def save_to_firestore(company_name, product_data):
    if not db:
        logging.error("Firestore ist nicht initialisiert.")
        return
    if product_data:
        try:
            doc_ref = db.collection("a").document(company_name)
            doc_ref.set({"products": product_data})
            logging.info(f"Produkte erfolgreich für {company_name} gespeichert.")
        except Exception as e:
            logging.error(f"Fehler beim Speichern in Firestore: {e}")
    else:
        logging.warning(f"Keine Produkte für {company_name} gefunden.")

# Funktion zum Scrapen und Speichern aller Produkte für einen Shop
def scrape_and_store_all_products(shop_data):
    try:
        base_url = shop_data['base_url']
        category_url = shop_data['category_url']
        selectors = shop_data['selectors']
        company_name = shop_data.get('company_name', 'Unknown Company')  # Manuell definierter Name
        
        if not company_name or company_name == 'Unknown Company':
            logging.warning("Unternehmensname fehlt oder ist unbekannt.")
        
        product_urls = get_product_urls(base_url, category_url, selectors)
        
        driver = webdriver.Chrome(options=options)
        product_data = {}
        for url in product_urls:
            product_details = scrape_product_details(driver, url, selectors)
            if product_details["name"]:
                product_data[product_details["name"]] = product_details
        
        save_to_firestore(company_name, product_data)
        driver.quit()
    except Exception as e:
        logging.error(f"Fehler beim Scrapen und Speichern von {shop_data['base_url']}: {e}")

# Liste der Shops mit ihren spezifischen Selektoren
shops = [
        {
        "base_url": "https://sys-temic.com",
        "category_url": "https://sys-temic.com/collections/all-products-1",
        "company_name": "Sys Temic",
        "selectors": {
            "product_block": "li.grid__item",  # Produktblock
            "product_link": "a.product-card-link",  # Link zum Produkt
            "product_name": "div.product__title h1.h3",  # Produkttitel
            "product_description": "div#template--22244440441097__main-description-description p.p1",  # Beschreibung des Produkts
            "image_gallery": "div#pageGallery-template--22244440441097__main",  # Angepasster Selektor für das Produktbild
            "size_options": "input[type='radio'][name='Size']",  # Größe Optionen
            "size_value_attr": "value",  # Wert der Größe
            "size_disabled_class": "disabled",  # Wenn die Größe deaktiviert ist
            "price": "div.price__regular > span.price-item--regular",  # Selektor für den regulären Preis
        }
    }
]



# Verwenden von ThreadPoolExecutor, um die Shops parallel zu scrapen
def scrape_multiple_shops(shops):
    with ThreadPoolExecutor(max_workers=3) as executor:
        executor.map(scrape_and_store_all_products, shops)

# Starten des Scraping-Prozesses für mehrere Shops
scrape_multiple_shops(shops)
