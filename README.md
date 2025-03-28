# School_project
from flask import Flask, render_template, request, redirect, url_for
from flask_sqlalchemy import SQLAlchemy
import pandas as pd
import plotly.express as px
import os

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///air_quality.db'
db = SQLAlchemy(app)

class AirQuality(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    city = db.Column(db.String(100), nullable=False)
    date = db.Column(db.String(10), nullable=False)
    pm25 = db.Column(db.Float, nullable=False)
    pm10 = db.Column(db.Float, nullable=False)
    co2 = db.Column(db.Float, nullable=False)
    lat = db.Column(db.Float, nullable=False)
    lon = db.Column(db.Float, nullable=False)

@app.route('/')
def index():
    return render_template('index.html', bootstrap=True)

@app.route('/upload', methods=['GET', 'POST'])
def upload():
    if request.method == 'POST':
        file = request.files['file']
        if file:
            filepath = os.path.join('uploads', file.filename)
            file.save(filepath)
            df = pd.read_csv(filepath)
            for _, row in df.iterrows():
                record = AirQuality(city=row['city'], date=row['date'], pm25=row['pm25'], pm10=row['pm10'], co2=row['co2'], lat=row['lat'], lon=row['lon'])
                db.session.add(record)
            db.session.commit()
            return redirect(url_for('dashboard'))
    return render_template('upload.html', bootstrap=True)

@app.route('/dashboard', methods=['GET', 'POST'])
def dashboard():
    query = AirQuality.query
    selected_city = request.form.get('city')
    if selected_city:
        query = query.filter_by(city=selected_city)
    
  data = query.all()
    df = pd.DataFrame([(d.city, d.date, d.pm25, d.pm10, d.co2, d.lat, d.lon) for d in data], 
                      columns=['city', 'date', 'pm25', 'pm10', 'co2', 'lat', 'lon'])
    
  fig_hist = px.histogram(df, x='pm25', title='PM2.5 Distribution')
    fig_line = px.line(df, x='date', y='pm25', color='city', title='PM2.5 Levels Over Time')
    fig_map = px.scatter_mapbox(df, lat='lat', lon='lon', color='pm25', size='pm25', 
                                hover_name='city', title='Air Quality Map', 
                                mapbox_style='open-street-map', zoom=3)
    
  graph_hist_html = fig_hist.to_html(full_html=False)
    graph_line_html = fig_line.to_html(full_html=False)
    graph_map_html = fig_map.to_html(full_html=False)
    
   cities = [d.city for d in AirQuality.query.distinct(AirQuality.city)]
    
   return render_template('dashboard.html', graph_hist_html=graph_hist_html, 
                           graph_line_html=graph_line_html, graph_map_html=graph_map_html, 
                           cities=cities, bootstrap=True)

if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(debug=True)

