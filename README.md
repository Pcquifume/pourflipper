Ceci est une aplication pour le rangement de photos ! Il a été réaliser sur Thonny puis convertie en exe avec auto_py_to_exe voici son code d'origine :

import os
import shutil
from tkinter import *
from tkinter import filedialog, messagebox, ttk
from PIL import Image, ImageTk
import glob
import datetime

def log_message(message):
    log_widget.config(state=NORMAL)
    log_widget.insert(END, message + '\n')
    log_widget.see(END)
    log_widget.config(state=DISABLED)

def copier_et_trier(dossier_source, log_widget, progress_var, compteur_widget):
    # Créer un dossier pour stocker les fichiers triés
    dossier_destination = os.path.join(os.path.dirname(dossier_source), "photos_triés")
    os.makedirs(dossier_destination, exist_ok=True)

    fichiers = glob.glob(os.path.join(dossier_source, '**/*'), recursive=True)
    total_fichiers = sum(1 for f in fichiers if os.path.isfile(f) and est_type_valide(f))

    compteur = 0

    for chemin_source in fichiers:
        if os.path.isfile(chemin_source) and est_type_valide(chemin_source):
            # Obtenir la date de prise de vue ou de modification
            try:
                img = Image.open(chemin_source)
                exif_data = img._getexif()
                if exif_data is not None and 36867 in exif_data:  # DateTimeOriginal
                    date_pris = exif_data[36867]
                    date_pris = datetime.datetime.strptime(date_pris, '%Y:%m:%d %H:%M:%S')
                else:
                    # Si pas de date EXIF, utiliser la date de modification
                    date_pris = datetime.datetime.fromtimestamp(os.path.getmtime(chemin_source))
            except Exception as e:
                log_message(f"Erreur lors de la lecture de {chemin_source}: {e}")
                continue

            mois_annee = date_pris.strftime("%m.%Y")  # Format MM.AAAA

            chemin_destination = os.path.join(dossier_destination, mois_annee)
            os.makedirs(chemin_destination, exist_ok=True)

            nom_fichier = os.path.basename(chemin_source)
            chemin_destination_final = os.path.join(chemin_destination, nom_fichier)

            if os.path.exists(chemin_destination_final):
                # Affiche la fenêtre d'aperçu pour les doublons
                montrer_aperçu(chemin_source, chemin_destination_final)
            else:
                try:
                    shutil.copy2(chemin_source, chemin_destination_final)
                    log_message(f"Copié: {nom_fichier} dans {mois_annee}")
                except Exception as e:
                    log_message(f"Erreur lors de la copie de {nom_fichier}: {e}")

            compteur += 1
            compteur_widget.config(text=f"Fichiers traités : {compteur}")
            progress_var.set((compteur / total_fichiers) * 100)

def est_type_valide(fichier):
    return any(fichier.endswith(ext) for ext, var in types_fichiers.items() if var.get() == 1)

def selectionner_dossier(fenetre, bouton_commencer, bouton_selection, log_widget, progress_var, compteur_widget):
    dossier = filedialog.askdirectory()
    if dossier:
        log_message(f"Dossier sélectionné: {dossier}")
        bouton_selection.pack_forget()
        bouton_commencer.config(state=NORMAL)
        bouton_commencer.dossier = dossier  # Stocker le dossier pour l'utiliser plus tard

def commencer_tri(dossier_source, log_widget, progress_var, compteur_widget):
    log_message("Début du tri...")
    copier_et_trier(dossier_source, log_widget, progress_var, compteur_widget)
    log_message("Tri terminé.")

def montrer_aperçu(chemin_fichier1, chemin_fichier2):
    apercu_fenetre = Toplevel()
    apercu_fenetre.title("Fichiers en Doublon")
    apercu_fenetre.geometry("800x400")

    def valider_choix():
        if keep1_var.get() and keep2_var.get():
            messagebox.showinfo("Choix", "Les deux fichiers seront conservés.")
        elif keep1_var.get():
            os.remove(chemin_fichier2)  # Supprimer le fichier 2
            log_message(f"{os.path.basename(chemin_fichier2)} a été supprimé.")
        elif keep2_var.get():
            os.remove(chemin_fichier1)  # Supprimer le fichier 1
            log_message(f"{os.path.basename(chemin_fichier1)} a été supprimé.")
        else:
            messagebox.showwarning("Choix", "Aucun fichier n'a été sélectionné.")

        apercu_fenetre.destroy()

    # Charger les images pour aperçu
    img1 = Image.open(chemin_fichier1).resize((200, 200))  # Enlever ANTIALIAS
    img2 = Image.open(chemin_fichier2).resize((200, 200))  # Enlever ANTIALIAS

    # Convertir en image Tkinter
    img1_tk = ImageTk.PhotoImage(img1)
    img2_tk = ImageTk.PhotoImage(img2)

    panel1 = Label(apercu_fenetre, image=img1_tk)
    panel1.image = img1_tk  # Garder une référence pour éviter que l'image soit garbage-collected
    panel1.grid(row=0, column=0, padx=10, pady=10)

    panel2 = Label(apercu_fenetre, image=img2_tk)
    panel2.image = img2_tk  # Garder une référence pour éviter que l'image soit garbage-collected
    panel2.grid(row=0, column=1, padx=10, pady=10)

    keep1_var = IntVar(value=1)
    keep2_var = IntVar(value=1)

    # Cases à cocher
    Checkbutton(apercu_fenetre, text=f"Conserver {os.path.basename(chemin_fichier1)}", variable=keep1_var).grid(row=1, column=0)
    Checkbutton(apercu_fenetre, text=f"Conserver {os.path.basename(chemin_fichier2)}", variable=keep2_var).grid(row=1, column=1)

    Button(apercu_fenetre, text="Valider", command=valider_choix).grid(row=2, columnspan=2, pady=10)

def creer_interface():
    global fenetre, log_widget, types_fichiers
    fenetre = Tk()
    fenetre.title("Tri de Photos")
    fenetre.geometry("800x600")  # Fenêtre plus grande
    fenetre.configure(bg="#2C2C2C")

    # Initialiser les variables IntVar pour les types de fichiers
    global types_fichiers
    types_fichiers = {
        '.jpg': IntVar(value=1),
        '.jpeg': IntVar(value=1),
        '.png': IntVar(value=1),
        '.gif': IntVar(value=1)
    }

    # Zone de log
    log_widget = Text(fenetre, height=15, width=70, bg="#1C1C1C", fg="#FFFFFF", font=("Arial", 10))
    log_widget.pack(pady=10)

    scrollbar = Scrollbar(fenetre, command=log_widget.yview, orient=VERTICAL)
    scrollbar.pack(side='right', fill='y')
    log_widget.config(yscrollcommand=scrollbar.set)

    # Frame pour les cases à cocher
    frame_fichiers = Frame(fenetre, bg="#2C2C2C")
    frame_fichiers.pack(pady=10)

    # Types de fichiers à trier
    for ext in types_fichiers.keys():
        Checkbutton(frame_fichiers, text=ext, variable=types_fichiers[ext], bg="#2C2C2C", fg="#FFFFFF", selectcolor="#2C2C2C").pack(side='left')

    # Compteur de fichiers traités
    compteur_widget = Label(fenetre, text="Fichiers traités : 0", font=("Arial", 14), bg="#2C2C2C", fg="#FFFFFF")
    compteur_widget.pack(pady=10)

    # Barre de progression
    progress_var = IntVar()
    progress_bar = ttk.Progressbar(fenetre, variable=progress_var, maximum=100)
    progress_bar.pack(pady=10, fill='x')

    # Bouton pour commencer le tri
    bouton_commencer = Button(fenetre, text="Commencer", state="disabled", command=lambda: commencer_tri(bouton_commencer.dossier, log_widget, progress_var, compteur_widget), font=("Arial", 14), bg="#28A745", fg="white")
    bouton_commencer.pack(pady=10)

    # Bouton pour sélectionner un dossier
    bouton_selection = Button(fenetre, text="Sélectionner un dossier", command=lambda: selectionner_dossier(fenetre, bouton_commencer, bouton_selection, log_widget, progress_var, compteur_widget), font=("Arial", 14), bg="#007BFF", fg="white")
    bouton_selection.pack(pady=10)

    # Bouton quitter
    bouton_quitter = Button(fenetre, text="Quitter", command=fenetre.quit, font=("Arial", 14), bg="#DC3545", fg="white")
    bouton_quitter.pack(pady=10)

    fenetre.mainloop()

if __name__ == "__main__":
    creer_interface()
