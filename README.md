from flask import Flask, request, render_template, redirect, url_for
from werkzeug.utils import secure_filename
from utils import extract_audio, transcribe_audio, clean_text, summarize_text, create_flashcards

app = Flask(__name__)
app.config['UPLOAD_FOLDER'] = 'uploads/'

if not os.path.exists('uploads'):
    os.makedirs('uploads')

flashcards_global = []

@app.route('/')
def index():
    return render_template('upload.html')

@app.route('/upload', methods=['POST'])
def upload_file():
    if 'file' not in request.files:
        return redirect(request.url)
    file = request.files['file']
    if file.filename == '':
        return redirect(request.url)
    if file:
        filename = secure_filename(file.filename)
        video_path = os.path.join(app.config['UPLOAD_FOLDER'], filename)
        file.save(video_path)
        
        audio_path = 'output_audio.wav'
        extract_audio(video_path, audio_path)
        
        transcript = transcribe_audio(audio_path)
        cleaned_text = clean_text(transcript)
        summary = summarize_text(cleaned_text)
        global flashcards_global
        flashcards_global = create_flashcards(summary)
        
        return render_template('index.html', flashcards=flashcards_global)

if __name__ == '__main__':
    import threading
    from telegram import Update
    from telegram.ext import Updater, CommandHandler, CallbackContext

    def start(update: Update, context: CallbackContext):
        update.message.reply_text('Hi! Use /flashcards to get the flashcards.')

    def send_flashcards(update: Update, context: CallbackContext):
        response = ""
        for flashcard in flashcards_global:
            response += f"Q: {flashcard['question']}\nA: {flashcard['answer']}\n\n"
        update.message.reply_text(response)

    def main_bot():
        updater = Updater("YOUR_TELEGRAM_API_TOKEN", use_context=True)
        dp = updater.dispatcher

        dp.add_handler(CommandHandler("start", start))
        dp.add_handler(CommandHandler("flashcards", send_flashcards))

        updater.start_polling()
        updater.idle()

    threading.Thread(target=main_bot).start()
    app.run(debug=True)
