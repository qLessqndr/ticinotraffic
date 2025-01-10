SUPSI 2024-25  
Data Visualization course, M-D3202E  
Teacher Giovanni Profeta


# Ticino's Traffic
Authors: [Sergio Fernandez Diz](https://github.com/SergioFernandezDiz), Mirko Keller, [Alessandro Carnio](https://github.com/qlessqndr)

[Ticino's Traffic](https://qlessqndr.github.io/ticinotraffic/ticinotraffic.html)

## 1. Introduction

Traffic congestion is a global issue that affects major cities and urban areas worldwide. With population growth and urban expansion, the number of vehicles on the roads has increased significantly over the past few decades. This phenomenon has led to a range of negative consequences, including traffic jams, air pollution, longer travel times, and an overall decrease in quality of life. The primary causes of this issue include rising household incomes, greater accessibility to cars, and urban planning that does not always optimally manage high traffic flows.

The consequences of traffic congestion are not limited to economic impacts but also have environmental and social effects. Air pollution caused by vehicle emissions and the noise from traffic negatively affect human health and overall well-being. Furthermore, increased traffic can lead to road accidents, impacting public safety.

### 1.1 Objective

The objective of this project is to analyze traffic patterns in Switzerland, specifically in Ticino. By examining traffic trends from 1963 to the present, both at the national level and within the Ticino region, we aim to identify areas with the highest traffic density, uncover key traffic patterns and to see how traffic change in the last 60 years. Our goal is to understand how traffic is distributed across the region, how roadways can be optimized, and to identify potential solutions for reducing traffic congestion. 

### 1.2 Research Questions

The research focuses on answering the following questions:
- What are the primary causes of traffic congestion in Ticino, and how have these causes evolved over time?
- How does traffic fluctuate throughout the year, and how is it affected during holiday periods?
- How have traffic trends in Ticino and Switzerland evolved over the past several decades and what are the worse places?
  

### 1.3 Target Readers

This research is aimed at urban planners, traffic management professionals and researchers interested in transportation systems. It will also be valuable to local authorities, environmental agencies focusing on reducing pollution and improving road safety and for normal people.

## 2. Dataset 
### 2.1 Dataset Source
For the implementation of the project, we are using a dataset obtained from an [official governmental website](https://www.astra.admin.ch/astra/en/home/documentation/data-and-information-products/traffic-data/data-and-publication/swiss-automatic-road-traffic-counts--sartc-/annual-and-monthly-results.html). The dataset consists of a total of 61 files, each covering a year from 1963 to the present. It provides data on the number of vehicles passing through over 400 traffic counting stations located throughout Switzerland, recorded over specific time periods.

### 2.2 Dataset Structure
As mentioned above, the dataset consists of a total of 61 files, each following different types of templates, which complicates the data reading process. One of the most common templates is as follows:

![image](https://github.com/user-attachments/assets/7d2e35e6-1790-4c8a-88d6-e9f84b5249f2)

As we can observe, for each station, there are a total of 5 different types of data:

- **ADT**: Average daily traffic
- **AWT**: Average weekday traffic
- **ADT Tu-Th**: Average daily traffic from Tuesday to Thursday
- **ADT Sa**: Average traffic on Saturdays
- **ADT Su**: Average traffic on Sundays
  
If needed, here is the [full legend](https://github.com/user-attachments/files/18332543/Bulletin_2023_en.xlsx.-.legend.pdf) provided:

### 2.3 Dataset Problems
As explained earlier, the dataset consists of a total of 61 files, of which 11 were provided in PDF format instead of Excel. This issue has made data processing much more complex, as the files cannot be read directly. Additionally, there is another challenge related to the stations, which are not fixed across all years from installation to the present. For instance, while certain stations may be present in 2020, it is not guaranteed that they will be available in 2021, making data analysis more complicated and limiting the types of analyses that can be performed.

## 3. Data pre-processing

### 3.1 Trasforming PDF Files into Tables
After requesting the complete dataset for all years via the email address available on the website, we received a total of 11 out of 61 files in PDF format. Below is an example:

![image](https://github.com/user-attachments/assets/4d4f25a1-5129-4703-a895-8e0bd8fab0dd)

To work with the data contained in these PDF files, the following steps were performed:

- PDFs were first converted into high-resolution images using [pdf2image](https://pypi.org/project/pdf2image/).
- Text was extracted from these images using OCR software [Tesseract](https://github.com/tesseract-ocr).
- From the extracted text, we focused on the MO-SO row, representing daily weekly mean traffic, to maintain consistency across datasets.

Due to the limitations of OCR, many numbers were extracted with errors, such as extra digits, missing digits, or misplaced values in the wrong cells. To address this, the extracted data was manually verified against the original PDFs, and corrections were made to ensure consistency and accuracy.

Below is the function used to extract text from a single image using Tesseract. This function is then used within **process_folder()** to process the entire folder containing all the images for a given year. As you can see, the process is performed using the **extract_table()** function, which returns the table within the image in the **Pandas_df** format.

```Python
def process_image(image_filename, image_folder, output_folder):
    image_path = os.path.join(image_folder, image_filename)
    logger.info(f"Processing image: {image_path}")

    try:
        table_df = extract_table(image_path)
        if not table_df.empty:
            output_csv = os.path.join(output_folder, f"{os.path.splitext(image_filename)[0]}.csv")
            table_df.to_csv(output_csv, index=False)
            logger.info(f"Saved table to {output_csv}")
        else:
            logger.warning(f"No data found in image: {image_filename}")
    except Exception as e:
        logger.error(f"Error processing {image_filename}: {str(e)}")

```
### 3.2 Standardizing Formats
After extracting tables from the various PDF files, several issues needed to be addressed.

One of the primary issues was the incorrect recognition of station names by the OCR used to extract text from the images. For example, some station names were misinterpreted and stored as: [COLOVRE>, GENÃ^VE, HÃ,,GENDO, ZAceRICH]. As evident, these errors stemmed from OCR misreadings of the station names.
To resolve this, given that each of the 11 files contained 400 rows, we utilized **thefuzz**, a Python library designed for fuzzy string matching. This library allows for comparison and matching of strings with minor discrepancies, such as spelling errors or slight variations. By comparing the OCR-extracted station names with the correct reference names, we were able to correct these errors and ensure consistency in the station names across all files.
Here is the code that was used:

```Python
# Iterate through each CSV file in the input folder
for filename in os.listdir(input_folder):
    if filename.endswith('.csv'):
        filepath = os.path.join(input_folder, filename)
        print(f'Processing {filename}...')

        # Load the CSV file
        df = pd.read_csv(filepath)

        if 'Name' not in df.columns:
            print(f"The file {filename} does not contain the 'Name' column. Skipping this file.")
            continue

        # Convert the 'Name' column to string and uppercase to ensure proper processing
        df['Name'] = df['Name'].astype(str).str.upper()

        # Apply name correction
        df['Name'] = df['Name'].apply(lambda x: correct_name(x, official_names, threshold=75))

        # Save the updated CSV file to the output folder
        output_path = os.path.join(output_folder, filename)
        df.to_csv(output_path, index=False,
                  encoding='utf-8-sig')  # Use 'utf-8-sig' to maintain compatibility with Excel
        print(f'{filename} updated and saved to {output_folder}')
```

Using this code, for each name in the file, the `process` and `fuzz` functions from the **thefuzz** library are applied to correct them.

In addition to this, a standard format has been set for the dataset to avoid inconsistencies across the different years.

![image](https://github.com/user-attachments/assets/55f75e34-99d1-4fea-9f6e-ac81e7c9dd86)


### 3.3 Handling Missing and Invalid Data
The handling of missing values was carried out following a structured approach to avoid introducing errors into the dataset:
- If the Annual Mean column was available for a station, missing monthly values were replaced with the annual mean.
- If the Annual Mean was unavailable, the mean of the existing monthly values for that station was calculated and used.

This method ensured that missing values were replaced with accurate approximations based on the available data.

### 3.4 Data Cleaning and Normalization
The OCR process introduced errors in numeric fields, including extra digits, missing digits, or misplaced values. These issues were identified and corrected by manually comparing the extracted data to the original PDFs. Decimal separators were standardized to use a period (.). Additionally, traffic counts from certain years were scaled by multiplying by 1,000 to align with datasets recorded in different units.

### 3.5 Dataset Integration
Once cleaned and standardized, all 60 years of data were merged into a single unified dataset. 
Consistency in column structures and naming conventions was ensured during integration. The 
merged dataset was validated against official reference metadata, such as station lists, to verify 
its accuracy.

### 3.6 Final Dataset Structure
The final dataset contained one row per traffic station with the following columns:
- `Location`: The name of the area where the station is located.
- `Measuring Station` : A unique identifier for each station.
- `Monthly Traffic Counts`: Columns from January to December recording traffic volumes for each month.
- `Annual Mean`: The yearly average traffic volume for each station.

### 3.7 Real-Time Dataset:

In addition to the process of making the main dataset usable, we have also created a real-time dataset that gets updated with new values every 30 minutes. This dataset contains the average speed at specific coordinates along roads and highways, allowing us to track real-time traffic conditions in the Ticino region. The dataset is generated using an API from [TomTom](https://www.tomtom.com/), which, when provided with coordinates, returns the average speed and the maximum speed for that segment of the road. You can also specify the precision radius to reduce the number of requests needed.

This real-time dataset is created through the following process:

1. **Initial Road Data**: An initial file containing road data, including coordinates, is downloaded using OpenStreetMap. This file includes specific road segments selected for traffic monitoring.

2. **Traffic Data Update**: Every 30 minutes, the file is reopened. For each line (road segment) within the file, the API is called to gather traffic information for that particular point. The information collected includes the average speed, maximum speed, and speed limits. The updated traffic data is then saved to a new file, which is continually updated with the latest traffic conditions.

With this dataset, we can visualize traffic information through a dynamic map like the one shown below:

![image](https://github.com/user-attachments/assets/a69ee2b6-0656-4ef5-b2e6-aa4950a1aa31)


### API Functionality:
The TomTom API provides real-time traffic data and allows you to track road conditions based on specific coordinates. The API response includes key traffic details, such as:

- **Average Speed**: The current speed at a specific point on the road.
- **Speed Limits**: The maximum allowed speed for the road segment.
- **Traffic Congestion**: The level of congestion, which is used to determine the current traffic condition.
- **Precision Radius**: This feature allows you to specify a radius around the given coordinates, helping to optimize the number of requests made.

### Dataset Creation Process:
1. **OpenStreetMap**: The initial road data is extracted using OpenStreetMap, which provides detailed map data, including the coordinates of road segments.
2. **API Integration**: After every 30 minutes, the dataset is updated by calling the TomTom API for each road segment, retrieving the current traffic speed, speed limits, and other relevant data.
3. **Saving Data**: The traffic information for each segment is collected and saved into a new file, providing an up-to-date snapshot of the road conditions.

### Code:
Here is the Python code that implements the process for collecting traffic data from the API and saving it to the dataset:

```python
def get_traffic_data(lat, lon):
    # Call the TomTom API to get the speed limits and traffic data
    geojson_data = get_speedbox.get_speed_limits(lat, lon, size)
    
    traffic_data_points = []
    
    # Loop through the returned geojson data
    for feature in geojson_data['features']:
        if feature['geometry']['type'] == 'LineString':
            coordinates = feature['geometry']['coordinates']
            
            # Get the speed limit from the feature properties
            speed_limit = feature['properties'].get('maxspeed', None)
            speed_limit = int(speed_limit) if speed_limit and speed_limit.isdigit() else 50
            
            # Get the center point of the road segment for traffic data
            center_index = len(coordinates) // 2
            center_point = (coordinates[center_index][1], coordinates[center_index][0])
            
            # Call the traffic API to get real-time traffic data at the center point
            traffic_info = traffic.get_data(center_point)
            
            # Get the current speed if available
            current_speed = float(traffic_info['current_speed']) if traffic_info and 'current_speed' in traffic_info else None
            
            # Append the traffic data point to the list
            traffic_data_points.append({
                'coordinates': coordinates,
                'speed_limit': speed_limit,
                'current_speed': current_speed,
                'color': get_color_by_congestion(50, speed_limit)
            })
    
    return traffic_data_points
```
## Data visualizations
In this project, a total of 6 visualizations were created, including maps, animations, bar plots, line plots, etc. To generate the various plots (bar plots/line plots), the JavaScript library **D3.js** was used. D3.js is a JavaScript library designed for creating interactive data visualizations on the web. It allows manipulation of HTML and SVG (Scalable Vector Graphics) documents based on dynamic data, enabling the creation of complex and interactive charts, such as diagrams, maps, scatter plots, histograms, and much more. The decision to use this library was made because it allows for fully custom visualizations, offering complete flexibility in design, though this increases the complexity compared to using tools that generate charts automatically.

For the animation aspect, **Unity** was used. Unity is a software development platform primarily used for creating video games, interactive simulations, augmented reality (AR) experiences, 3D applications, and much more. Unity was chosen due to its robust capabilities in creating animations, offering a high degree of control over the visuals.

### 1. Switzerland's Traffic

![image](https://github.com/user-attachments/assets/ebf98039-bf14-4f75-a59e-aa624d639f03)

#### 1.1 Brief Description
This animation displays the daily average number of cars passing through various points within a specific region. Switzerland has been divided into 16 zones to prevent data visualization overload. This chart allows for the analysis of the average vehicle flow across different regions of Switzerland from 1963 to 2024. To generate this visualization, data from over 400 traffic counting stations scattered across the country's main roads and highways are used.

The animation provides a clear view of how traffic patterns have evolved over the years in each zone, offering valuable insights into regional traffic dynamics and trends. Furthermore, it allows for the analysis of how traffic is influenced by various occurrences, such as crises, pandemics, and other significant events.

#### 1.2 Research Question
How has traffic flow evolved in Switzerland, and how is it affected by crises and pandemics?

#### 1.3 Type of Visualization: 
Animation with map plot + line plot

#### 1.4 Involved variables:
Map Plot
- Numerical Variable, Color and Size -> Daily average number of cars passing through a traffic counting station
- Coordinates/Location -> Canton/Region of Switzerland
  
Line Plot
- X-axis -> Year from 1963 to 2023
- Y-axis -> Daily average number of vehicles passing through all the stations in Switzerland

### 2. Top busiest stations in Ticino

![image](https://github.com/user-attachments/assets/baaeaad9-8e76-4e13-9e9d-5613d397035c)

#### 2.1 Brief Description
This visualization shows the 6 busiest traffic counting stations in Ticino from 2003 to 2023. Additionally, by using a map to display the locations of these 6 stations in Ticino, we can analyze traffic patterns more specifically within the region. This allows us to identify patterns in traffic coming from the central part of Switzerland or from Italy. The visualization helps us understand whether the main source of traffic in Ticino comes from cross-border workers (frontaliers), local residents, or people traveling from the San Gottardo region (central Switzerland).

#### 2.2 Research Question
How has traffic in Ticino evolved from 2003 to 2023? Furthermore, is the traffic problem in Ticino solely due to the border with Italy, or is it also influenced by traffic from central Switzerland?

#### 2.3 Type of Visualization: 
Animated Bar Plot + Map Plot

#### 2.4 Involved Variables:

**Map Plot**
- Hover(tooltip) -> Name of Station, Daily Vehicle Count
- Coordinates/Location -> Station Position
- Color/Size -> Daily Vehicle Count and position in the top 6 (`gray` = not present in the top)

**Bar Plot:**
- X-axis: Number of daily vehicle counts 
- Y-axis: Top 6 counting stations in Ticino
- Color: Number of daily vehicle counts
- Position: Shows the position in the Top 6

### 3. Ticino's Traffic

![image](https://github.com/user-attachments/assets/3a0596b8-5583-4de5-88d7-dc1672ac96ae)

#### 3.1 Brief Description
This visualization shows the traffic trends in Ticino from 1963 to 2023, displaying patterns and how they were influenced by phenomena such as COVID and economic crises.

#### 3.2 Research Question
How has traffic in Ticino evolved from 1963 to 2023? What have been the most significant increases and decreases in traffic?

#### 3.3 Type of Visualization: 
Line Plot

#### 3.4 Involved Variables:
**Line Plot:**
- X-axis: Year from 1963 to 2023
- Y-axis: Average number of vehicles passing through the counting stations
- Hover(tooltip): Exact value of vehicles and year


### 4. Ticino's Monthly Patterns

![image](https://github.com/user-attachments/assets/e7329ddc-6704-47c3-a2a7-6251406c5a6f)

#### 4.1 Brief Description
This visualization shows the average daily vehicle count sampled for each month in the year 2023. Since the overall trend is quite linear, the top 6 counting stations are also included to allow comparison and see if certain areas are more influenced by holidays or festivities, while others may be affected by different factors.

#### 4.2 Research Question
What are the main traffic patterns throughout the year in Ticino? How is traffic volume affected during holiday periods?

#### 4.3 Type of Visualization: 
Line Plot

#### 4.4 Involved Variables:
**Line Plot:**
- X-axis: Month of the year
- Y-axis: Average number of vehicles passing through the counting stations daily
- Color: Station Name


### 5. Real-Time Traffic Map

![image](https://github.com/user-attachments/assets/f353665a-ef40-4012-8858-4dfaede4474b)


#### 5.1 Brief Description
This map is created using the TomTom API and shows the average speed on a selected set of roads and highways in Ticino. The roads are color-coded from green to red based on the average speed compared to the maximum allowed speed for each segment. This visualization allows the user to select a historical day or view real-time traffic with updates every 30 minutes.

#### 5.2 Research Question
When thinking about traffic, is the problem always caused by the presence of vehicles, or could factors like the number of traffic lights in urban areas also contribute? How is the traffic in the main areas of Ticino on a daily basis?

#### 5.3 Type of Visualization: 
Real-Time Map Plot

#### 5.4 Involved Variables:

**Map Plot:**
- **Map**: A map of Ticino showing the main roads and cities.
- **Color**: The color gradient, from green to red, indicates how the average speed compares to the maximum allowed speed on each segment of the road.
- **Scrollbar**: Allows the user to select a specific time on a given day, highlighting the differences in traffic patterns between morning, afternoon, and evening.

## Key findings

Through this analysis, we have observed that both at the national level in Switzerland and at the Ticino level, the issue of traffic congestion continues to rise. We have also noticed that traffic in Ticino is heavily influenced by the presence of both tourists and cross-border workers, as well as by significant events like the Covid-19 pandemic.

In terms of annual trends, we observed a sharp increase in traffic between June and August, largely due to the vacation period. This influx is driven by both local residents traveling and the many tourists who visit Ticino during the summer months.

As for the most congested roads, the areas around Lugano and the A2 motorway, which leads to the San Gottardo Pass, experience the highest traffic volumes. The A2 serves as a key route for both daily commuters and tourists, contributing to its heavy traffic.

## Next Steps

During the execution of this project, we encountered several issues and limitations due to the dataset. Here are the main Next steps :

- Due to the size of the different visualizations (**video** and **Real-Time Map**), we faced **loading issues**. We had difficulties creating the animation with higher resolution and also encountered problems with loading the **Real-Time Map**. One possible next step would be to resolve this issue by developing the project on a website that allows for better performance and smoother loading.
  
- Another important aspect would be to conduct a more **detailed analysis**, perhaps by finding a dataset that provides information about the **direction** in which the vehicles are moving. In our case, the stations only provided data on the vehicles passing through a specific place, but this wasn't sufficient for a more precise analysis of the traffic movements to determine the exact reasons behind the traffic volumes.

- Regarding the **Real-Time Map**, which was created using a free **TomTom API**, it would be interesting to make it more detailed, offering a broader view. To do this, we would need another **paid API** that offers more daily requests and more detailed data.

