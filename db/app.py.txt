import pandas as pd
import datetime as dt
import os
from dash import dcc, html
from dash.dependencies import Input, Output
import dash
import plotly.graph_objects as go
import tab1 
import tab2  
import tab3  

class db:
        self.merged = df
        
df = db()
df.merge()

external_stylesheets = ['https://codepen.io/chriddyp/pen/bWLwgP.css']
app = dash.Dash(__name__, external_stylesheets=external_stylesheets, suppress_callback_exceptions=True)

app.layout = html.Div(
    [
        html.Div(
            [
                dcc.Tabs(
                    id='tabs',
                    value='tab-1',
                    children=[
                        dcc.Tab(label='Sprzedaż globalna', value='tab-1'),
                        dcc.Tab(label='Produkty', value='tab-2'),
                        dcc.Tab(label='Kanały sprzedaży', value='tab-3'),
                    ],
                ),
                html.Div(id='tabs-content'),
            ],
            style={'width': '80%', 'margin': 'auto'},
        )
    ],
    style={'height': '100%'},
)

@app.callback(Output('tabs-content', 'children'), [Input('tabs', 'value')])
def render_content(tab):
    if tab == 'tab-1':
        return tab1.render_tab(df.merged)
    elif tab == 'tab-3':
        return tab3.render_tab(df.merged)
    elif tab == 'tab-2':
        return tab2.render_tab(df.merged)
        
## tab1 callbacks
@app.callback(Output('bar-sales','figure'),
    [Input('sales-range','start_date'),Input('sales-range','end_date')])

def tab1_bar_sales(start_date,end_date):

    truncated = df.merged[(df.merged['tran_date']>=start_date)&(df.merged['tran_date']<=end_date)]
    grouped = truncated[truncated['total_amt']>0].groupby([pd.Grouper(key='tran_date',freq='M'),'Store_type'])['total_amt'].sum().round(2).unstack()

    traces = []
    for col in grouped.columns:
        traces.append(go.Bar(x=grouped.index,y=grouped[col],name=col,hoverinfo='text',
        hovertext=[f'{y/1e3:.2f}k' for y in grouped[col].values]))

    data = traces
    fig = go.Figure(data=data,layout=go.Layout(title='Przychody',barmode='stack',legend=dict(x=0,y=-0.5)))

    return fig
    
@app.callback(Output('choropleth-sales','figure'),
            [Input('sales-range','start_date'),Input('sales-range','end_date')])
def tab1_choropleth_sales(start_date,end_date):

    truncated = df.merged[(df.merged['tran_date']>=start_date)&(df.merged['tran_date']<=end_date)]
    grouped = truncated[truncated['total_amt']>0].groupby('country')['total_amt'].sum().round(2)

    trace0 = go.Choropleth(colorscale='Viridis',reversescale=True,
                            locations=grouped.index,locationmode='country names',
                            z = grouped.values, colorbar=dict(title='Sales'))
    data = [trace0]
    fig = go.Figure(data=data,layout=go.Layout(title='Mapa',geo=dict(showframe=False,projection={'type':'natural earth'})))

    return fig

## tab2 callbacks
@app.callback(Output('barh-prod-subcat','figure'),
            [Input('prod_dropdown','value')])
def tab2_barh_prod_subcat(chosen_cat):

    grouped = df.merged[(df.merged['total_amt']>0)&(df.merged['prod_cat']==chosen_cat)].pivot_table(index='prod_subcat',columns='Gender',values='total_amt',aggfunc='sum').assign(_sum=lambda x: x['F']+x['M']).sort_values(by='_sum').round(2)

    traces = []
    for col in ['F','M']:
        traces.append(go.Bar(x=grouped[col],y=grouped.index,orientation='h',name=col))

    data = traces
    fig = go.Figure(data=data,layout=go.Layout(barmode='stack',margin={'t':20,}))
    return fig

## tab3 callbacks
@app.callback(Output('bar-store', 'figure'),
              [Input('prod_dropdown', 'value')])
def tab3_bar_store(chosen_cat):
    grouped = (
        df.merged[
            (df.merged['total_amt'] > 0) & (df.merged['Store_type'] == chosen_cat)
        ]
        .pivot_table(index='day_of_week', columns='Store_type', values='total_amt', aggfunc='sum')
    )

    traces = []
    for col in grouped.columns:
        traces.append(
            go.Bar(
                x=grouped[col],  
                y=grouped.index, 
                orientation='h',
                name=col
            )
        )

    fig = go.Figure(
        data=traces,
        layout=go.Layout(
            barmode='stack',
            margin={'t': 40},
            yaxis_title='Dzień tygodnia',
            xaxis_title='Sprzedaż',
        )
    )

    return fig

@app.callback(
    Output('bar-age-gender', 'figure'),
    [Input('prod_dropdown', 'value')]  # Using the same dropdown to filter by Store_type
)
def tab3_bar_age_gender(chosen_cat):
    filtered_data = df.merged[df.merged['Store_type'] == chosen_cat]
    
    grouped = filtered_data.groupby(['Store_type', 'age', 'Gender']).size().reset_index(name='count')

    fig = go.Figure()

    for gender in grouped['Gender'].unique():
        gender_data = grouped[grouped['Gender'] == gender]
        fig.add_trace(go.Bar(
            x=gender_data['age'],
            y=gender_data['count'],
            name=gender,
            orientation='v'
        ))

    fig.update_layout(
        barmode='group',
        title='Kanały sprzedaży ze wzg na wiek i płeć',
        xaxis_title='Wiek',
        yaxis_title='Ilość',
        xaxis=dict(tickmode='linear'),
        showlegend=True
    )

    return fig


if __name__ == '__main__':
    app.run_server(debug=True)
